
USE ALM;
SET NOCOUNT ON;

DECLARE @dt_rep date = '2026-09-01';
DECLARE @d_end  date = '2029-04-30';

DROP TABLE IF EXISTS #base;
DROP TABLE IF EXISTS #model;
DROP TABLE IF EXISTS #numbers;
DROP TABLE IF EXISTS #payments;
DROP TABLE IF EXISTS #daily_payments;
DROP TABLE IF EXISTS #init;
DROP TABLE IF EXISTS #calendar;


/* 1. Берём фактический ОСЗ на дату среза.
   Группировка только по IS_FLOATRATE. */

SELECT
    ROW_NUMBER() OVER (
        ORDER BY t.CON_ID, t.DT_CLOSE
    ) AS loan_id,

    ISNULL(TRY_CAST(t.IS_FLOATRATE AS int), 0) AS is_floatrate,
    CAST(t.DT_CLOSE AS date) AS dt_close_d,
    ABS(CAST(t.OUT_RUB AS float)) AS out_rub,
    CAST(t.RATE_CON AS float) AS rate_int,

    DATEDIFF(month, @dt_rep, CAST(t.DT_CLOSE AS date))
    + CASE
        WHEN @dt_rep < EOMONTH(@dt_rep) THEN 1
        ELSE 0
      END AS n_payments

INTO #base
FROM ALM.ALM.balance_rest_all t WITH (NOLOCK)
WHERE t.DT_REP = @dt_rep
  AND t.BLOCK_NAME = N'Кредиты ФЛ'
  AND t.SECTION_NAME = N'Ипотека'
  AND t.OD_FLAG = 1
  AND t.DT_CLOSE >= DATEADD(day, 1, @dt_rep)
  AND t.OUT_RUB IS NOT NULL
  AND t.OUT_RUB <> 0;

CREATE UNIQUE CLUSTERED INDEX IX_base
ON #base (loan_id);

IF EXISTS (
    SELECT 1
    FROM #base
    WHERE rate_int < 0 OR rate_int > 1
)
    THROW 50001, 'RATE_CON должна быть в долях единицы', 1;


/* 2. Рассчитываем аннуитет от текущего ОСЗ
   на оставшийся срок до DT_CLOSE.

   Первый и последний неполные периоды учитываем
   по дням, обычные месяцы — RATE_CON / 12. */

DECLARE @FirstMonthOffset int =
    CASE
        WHEN @dt_rep = EOMONTH(@dt_rep) THEN 1
        ELSE 0
    END;

;WITH periods AS (
    SELECT
        b.*,

        ISNULL(b.rate_int, 0) / 12.0 AS i_month,

        ISNULL(b.rate_int, 0)
        * DATEDIFF(
            day,
            @dt_rep,
            CASE
                WHEN b.dt_close_d <
                     EOMONTH(@dt_rep, @FirstMonthOffset)
                    THEN b.dt_close_d
                ELSE EOMONTH(@dt_rep, @FirstMonthOffset)
            END
        ) / 365.0 AS i_first,

        CASE
            WHEN b.n_payments = 1 THEN 0.0

            WHEN b.dt_close_d = EOMONTH(b.dt_close_d)
                THEN ISNULL(b.rate_int, 0) / 12.0

            ELSE
                ISNULL(b.rate_int, 0)
                * DATEDIFF(
                    day,
                    EOMONTH(
                        @dt_rep,
                        @FirstMonthOffset + b.n_payments - 2
                    ),
                    b.dt_close_d
                ) / 365.0
        END AS i_last

    FROM #base b
),

factors AS (
    SELECT
        p.*,
        POWER(
            1.0 + p.i_month,
            -CAST(p.n_payments - 2 AS float)
        ) AS q
    FROM periods p
),

pv AS (
    SELECT
        f.*,

        CASE
            WHEN f.n_payments = 1
                THEN 1.0 / (1.0 + f.i_first)

            ELSE
                1.0 / (1.0 + f.i_first)
                *
                (
                    1.0
                    + CASE
                        WHEN f.n_payments <= 2 THEN 0.0

                        WHEN ABS(f.i_month) < 0.000000000001
                            THEN CAST(f.n_payments - 2 AS float)

                        ELSE (1.0 - f.q) / f.i_month
                      END
                    + f.q / (1.0 + f.i_last)
                )
        END AS pv_factor

    FROM factors f
)

SELECT
    loan_id,
    is_floatrate,
    out_rub,
    rate_int,
    dt_close_d,
    n_payments,
    i_month,
    i_first,
    i_last,

    CASE
        WHEN rate_int IS NULL THEN 0.0
        ELSE out_rub / NULLIF(pv_factor, 0)
    END AS annuity_payment

INTO #model
FROM pv;

CREATE UNIQUE CLUSTERED INDEX IX_model
ON #model (loan_id);


/* 3. Создаём номера платежей до конца горизонта. */

DECLARE @ProjectedMonths int =
    DATEDIFF(month, @dt_rep, @d_end) + 2;

SELECT TOP (@ProjectedMonths)
    CAST(
        ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1
        AS int
    ) AS n

INTO #numbers
FROM sys.all_objects a
CROSS JOIN sys.all_objects b;

CREATE UNIQUE CLUSTERED INDEX IX_numbers
ON #numbers (n);


/* 4. Моделируем ежемесячные платежи.
   Проценты = ОСЗ до платежа * ставка за период.
   Погашение тела = аннуитет - проценты.
   В DT_CLOSE погашаем весь оставшийся долг. */

;WITH schedule AS (
    SELECT
        m.*,
        n.n,

        CASE
            WHEN n.n = m.n_payments - 1
                THEN m.dt_close_d
            ELSE EOMONTH(
                @dt_rep,
                @FirstMonthOffset + n.n
            )
        END AS payment_date,

        CASE
            WHEN n.n = 0 THEN m.i_first
            WHEN n.n = m.n_payments - 1 THEN m.i_last
            ELSE m.i_month
        END AS period_rate

    FROM #model m
    INNER JOIN #numbers n
        ON n.n < m.n_payments
),

previous_balance AS (
    SELECT
        p.*,

        CASE
            WHEN p.n = 0 THEN p.out_rub

            ELSE
                (
                    p.out_rub * (1.0 + p.i_first)
                    - p.annuity_payment
                )
                * POWER(1.0 + p.i_month, p.n - 1)

                - p.annuity_payment
                * CASE
                    WHEN ABS(p.i_month) < 0.000000000001
                        THEN CAST(p.n - 1 AS float)
                    ELSE
                        (
                            POWER(1.0 + p.i_month, p.n - 1)
                            - 1.0
                        ) / p.i_month
                  END
        END AS balance_before_raw

    FROM schedule p
    WHERE p.payment_date <= @d_end
),

balances AS (
    SELECT
        p.*,
        CASE
            WHEN p.balance_before_raw < 0 THEN 0.0
            ELSE p.balance_before_raw
        END AS balance_before
    FROM previous_balance p
),

interest_calc AS (
    SELECT
        b.*,
        b.balance_before * b.period_rate
            AS interest_payment
    FROM balances b
),

payments AS (
    SELECT
        i.*,

        CASE
            WHEN i.n = i.n_payments - 1
                THEN i.balance_before

            WHEN i.annuity_payment - i.interest_payment <= 0
                THEN 0.0

            WHEN i.annuity_payment - i.interest_payment
                 >= i.balance_before
                THEN i.balance_before

            ELSE i.annuity_payment - i.interest_payment
        END AS principal_payment

    FROM interest_calc i
)

SELECT
    loan_id,
    is_floatrate,
    payment_date AS d,
    rate_int,

    CAST(principal_payment AS decimal(28,6))
        AS principal_payment,

    CAST(interest_payment AS decimal(28,6))
        AS interest_payment

INTO #payments
FROM payments
OPTION (MAXDOP 2, RECOMPILE);

CREATE CLUSTERED INDEX IX_payments
ON #payments (is_floatrate, d);


/* 5. Агрегируем платежи по датам и IS_FLOATRATE.
   Сохраняем веса для пересчёта клиентской ставки. */

SELECT
    is_floatrate,
    d,

    SUM(principal_payment) AS principal_payment,
    SUM(interest_payment) AS interest_payment,

    SUM(CASE
        WHEN rate_int IS NOT NULL
            THEN principal_payment * rate_int
        ELSE 0
    END) AS principal_int_num,

    SUM(CASE
        WHEN rate_int IS NOT NULL
            THEN principal_payment
        ELSE 0
    END) AS principal_int_den

INTO #daily_payments
FROM #payments
GROUP BY is_floatrate, d;

CREATE UNIQUE CLUSTERED INDEX IX_daily_payments
ON #daily_payments (is_floatrate, d);


/* 6. Исходные ОСЗ и средневзвешенные ставки. */

SELECT
    is_floatrate,
    SUM(out_rub) AS init_out,

    SUM(CASE
        WHEN rate_int IS NOT NULL
            THEN out_rub * rate_int
        ELSE 0
    END) AS init_int_num,

    SUM(CASE
        WHEN rate_int IS NOT NULL
            THEN out_rub
        ELSE 0
    END) AS init_int_den

INTO #init
FROM #base
GROUP BY is_floatrate;

CREATE UNIQUE CLUSTERED INDEX IX_init
ON #init (is_floatrate);


/* 7. Подневной календарь строго с @dt_rep.
   На первый день приходится исходный фактический ОСЗ. */

SELECT TOP (DATEDIFF(day, @dt_rep, @d_end) + 1)
    DATEADD(
        day,
        CAST(
            ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1
            AS int
        ),
        @dt_rep
    ) AS d

INTO #calendar
FROM sys.all_objects a
CROSS JOIN sys.all_objects b;

CREATE UNIQUE CLUSTERED INDEX IX_calendar
ON #calendar (d);


/* 8. Прогнозируем подневной ОСЗ.
   Вычитаем накопленные погашения тела.
   Проценты не уменьшают ОСЗ.
   Ставку взвешиваем по оставшемуся долгу. */

;WITH grid AS (
    SELECT
        i.is_floatrate,
        c.d,

        i.init_out,
        i.init_int_num,
        i.init_int_den,

        ISNULL(p.principal_payment, 0)
            AS principal_payment,

        ISNULL(p.interest_payment, 0)
            AS interest_payment,

        ISNULL(p.principal_int_num, 0)
            AS principal_int_num,

        ISNULL(p.principal_int_den, 0)
            AS principal_int_den

    FROM #init i
    CROSS JOIN #calendar c

    LEFT JOIN #daily_payments p
        ON p.is_floatrate = i.is_floatrate
       AND p.d = c.d
),

cumulative AS (
    SELECT
        g.*,

        SUM(principal_payment) OVER (
            PARTITION BY is_floatrate
            ORDER BY d
            ROWS UNBOUNDED PRECEDING
        ) AS cum_principal,

        SUM(principal_int_num) OVER (
            PARTITION BY is_floatrate
            ORDER BY d
            ROWS UNBOUNDED PRECEDING
        ) AS cum_int_num,

        SUM(principal_int_den) OVER (
            PARTITION BY is_floatrate
            ORDER BY d
            ROWS UNBOUNDED PRECEDING
        ) AS cum_int_den

    FROM grid g
)

/* Итог: один сценарий, группировка по IS_FLOATRATE. */

SELECT
    d AS [date],
    is_floatrate,

    CAST(
        CASE
            WHEN init_out - cum_principal < 0 THEN 0
            ELSE init_out - cum_principal
        END
        AS decimal(28,2)
    ) AS out_rub,

    CAST(
        (init_int_num - cum_int_num)
        / NULLIF(init_int_den - cum_int_den, 0)
        AS decimal(12,6)
    ) AS rate_int_srvz,

    CAST(interest_payment AS decimal(28,2))
        AS interest_payment,

    CAST(principal_payment AS decimal(28,2))
        AS principal_payment,

    CAST(
        interest_payment + principal_payment
        AS decimal(28,2)
    ) AS total_payment

FROM cumulative
ORDER BY d, is_floatrate
OPTION (RECOMPILE);
