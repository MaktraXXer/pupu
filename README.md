
USE ALM;
SET NOCOUNT ON;

DECLARE @dt_rep date = '2026-09-01';
DECLARE @d_end  date = '2029-04-30';

DROP TABLE IF EXISTS #base;
DROP TABLE IF EXISTS #model0;
DROP TABLE IF EXISTS #model;
DROP TABLE IF EXISTS #numbers;
DROP TABLE IF EXISTS #payments;
DROP TABLE IF EXISTS #daily_payments;
DROP TABLE IF EXISTS #init;
DROP TABLE IF EXISTS #calendar;
DROP TABLE IF EXISTS #daily;


/* ============================================================
   1. ИСХОДНЫЙ ИПОТЕЧНЫЙ ПОРТФЕЛЬ

   Каждая строка баланса рассматривается отдельно.

   DT_OPEN_FACT используется во втором сценарии.

   Если RATE_CON отсутствует:
   - не предполагаем неизвестную процентную ставку;
   - плановая амортизация не моделируется;
   - остаток полностью погашается в DT_CLOSE.
   ============================================================ */

;WITH mapping AS (
    SELECT
        prod_name,
        MAX(AGG_PROD_NAME) AS AGG_PROD_NAME
    FROM ALM_TEST.mort.man_mort_mapping WITH (NOLOCK)
    GROUP BY prod_name
)

SELECT
    ROW_NUMBER() OVER (
        ORDER BY t.CON_ID, t.DT_CLOSE
    ) AS loan_id,

    COALESCE(
        CAST(b.AGG_PROD_NAME AS nvarchar(255)),
        N'Без классификации'
    ) AS is_floatrate,

    CAST(t.DT_OPEN_FACT AS date) AS dt_open_fact,
    CAST(t.DT_CLOSE AS date) AS dt_close_d,

    CAST(ABS(t.OUT_RUB) AS float) AS out_rub,

    CAST(t.RATE_CON AS float) AS rate_int,
    CAST(t.RATE_TRF AS float) AS rate_trf,

    DATEDIFF(
        month,
        @dt_rep,
        CAST(t.DT_CLOSE AS date)
    )
    + CASE
        WHEN @dt_rep < EOMONTH(@dt_rep) THEN 1
        ELSE 0
      END AS n_payments

INTO #base

FROM ALM.ALM.balance_rest_all t WITH (NOLOCK)

LEFT JOIN mapping b
    ON b.prod_name = t.PROD_NAME_RES

WHERE t.DT_REP = @dt_rep
  AND t.BLOCK_NAME = N'Кредиты ФЛ'
  AND t.SECTION_NAME = N'Ипотека'
  AND t.OD_FLAG = 1
  AND CAST(t.DT_CLOSE AS date) > @dt_rep
  AND t.OUT_RUB IS NOT NULL
  AND t.OUT_RUB <> 0;

CREATE UNIQUE CLUSTERED INDEX IX_base
ON #base (loan_id);

CREATE INDEX IX_base_group
ON #base (is_floatrate);


/* Проверка единиц измерения ставки */

IF EXISTS (
    SELECT 1
    FROM #base
    WHERE rate_int < 0 OR rate_int > 1
)
    THROW 50001,
        'RATE_CON должен быть в долях единицы: 15% = 0.15',
        1;


/* ============================================================
   2. ДВА СЦЕНАРИЯ АННУИТЕТА

   Метод 1:
       Проценты первого периода начисляем с @dt_rep.

   Метод 2:
       Первый процентный период начинается с предыдущего
       конца месяца либо DT_OPEN_FACT, если выдача позже.

   В обоих случаях исходный долг = фактический OUT_RUB.
   ============================================================ */

DECLARE @FirstMonthOffset int =
    CASE
        WHEN @dt_rep = EOMONTH(@dt_rep) THEN 1
        ELSE 0
    END;

SELECT
    b.loan_id,
    b.is_floatrate,
    b.out_rub,
    b.rate_int,
    b.rate_trf,
    b.dt_close_d,
    b.n_payments,

    v.method_id,

    f.first_payment_date,

    CAST(ISNULL(b.rate_int, 0) AS float) / 12.0
        AS i_month,

    CAST(ISNULL(b.rate_int, 0) AS float)
        * DATEDIFF(day, a.interest_start, f.first_payment_date)
        / 365.0 AS i_first,

    CASE
        WHEN b.n_payments = 1 THEN 0.0

        WHEN b.dt_close_d = EOMONTH(b.dt_close_d)
            THEN CAST(ISNULL(b.rate_int, 0) AS float) / 12.0

        ELSE
            CAST(ISNULL(b.rate_int, 0) AS float)
            * DATEDIFF(
                day,
                EOMONTH(
                    @dt_rep,
                    @FirstMonthOffset + b.n_payments - 2
                ),
                b.dt_close_d
              ) / 365.0
    END AS i_last

INTO #model0

FROM #base b

CROSS JOIN (VALUES (1), (2)) v(method_id)

CROSS APPLY (
    SELECT
        CASE
            WHEN b.dt_close_d <
                 EOMONTH(@dt_rep, @FirstMonthOffset)
                THEN b.dt_close_d
            ELSE EOMONTH(@dt_rep, @FirstMonthOffset)
        END AS first_payment_date
) f

CROSS APPLY (
    SELECT
        CASE
            WHEN v.method_id = 1
                THEN @dt_rep

            WHEN @dt_rep = EOMONTH(@dt_rep)
                THEN @dt_rep

            WHEN b.dt_open_fact IS NULL
              OR b.dt_open_fact > @dt_rep
                THEN @dt_rep

            WHEN b.dt_open_fact > EOMONTH(@dt_rep, -1)
                THEN b.dt_open_fact

            ELSE EOMONTH(@dt_rep, -1)
        END AS interest_start
) a;

CREATE UNIQUE CLUSTERED INDEX IX_model0
ON #model0 (loan_id, method_id);


/* ============================================================
   3. РАССЧИТЫВАЕМ РАЗМЕР ЕЖЕМЕСЯЧНОГО АННУИТЕТА

   Используем PV всех оставшихся платежей.

   Первый и последний неполные периоды учитываются
   отдельно, обычные месяцы — RATE_CON / 12.

   Уровень аннуитета определяется на весь оставшийся
   срок кредита, даже если он заканчивается после @d_end.
   ============================================================ */

;WITH factors AS (
    SELECT
        m.*,

        POWER(
            1.0 + m.i_month,
            -CAST(m.n_payments - 2 AS float)
        ) AS q_power

    FROM #model0 m
),

annuity_factors AS (
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

                        ELSE
                            (1.0 - f.q_power) / f.i_month
                      END

                    + f.q_power / (1.0 + f.i_last)
                )
        END AS annuity_factor

    FROM factors f
)

SELECT
    f.loan_id,
    f.is_floatrate,
    f.out_rub,
    f.rate_int,
    f.rate_trf,
    f.dt_close_d,
    f.n_payments,
    f.method_id,

    f.i_month,
    f.i_first,
    f.i_last,

    CASE
        WHEN f.rate_int IS NULL THEN 0.0
        ELSE f.out_rub / NULLIF(f.annuity_factor, 0)
    END AS annuity_payment

INTO #model

FROM annuity_factors f;

CREATE UNIQUE CLUSTERED INDEX IX_model
ON #model (loan_id, method_id);


/* ============================================================
   4. НОМЕРА БУДУЩИХ МЕСЯЧНЫХ ПЛАТЕЖЕЙ

   Генерируем только платежи до @d_end.

   При этом размер аннуитета уже рассчитан
   на полный оставшийся срок кредита.
   ============================================================ */

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


/* ============================================================
   5. ПРОЦЕНТНЫЕ И ОСНОВНЫЕ ПЛАТЕЖИ

   Для каждого платежа:

   interest = ОСЗ_до_платежа * ставка_за_период

   principal = annuity - interest

   На DT_CLOSE:
       principal = весь оставшийся ОСЗ

   Все погашения ограничены текущим ОСЗ.
   ============================================================ */

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
        END AS payment_date

    FROM #model m

    INNER JOIN #numbers n
        ON n.n < m.n_payments
),

periods AS (
    SELECT
        s.*,

        CASE
            WHEN s.n = 0 THEN s.i_first

            WHEN s.n = s.n_payments - 1
                THEN s.i_last

            ELSE s.i_month
        END AS period_rate

    FROM schedule s

    WHERE s.payment_date <= @d_end
),

previous_balance AS (
    SELECT
        p.*,

        CASE
            WHEN p.n = 0
                THEN p.out_rub

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

    FROM periods p
),

interest_calc AS (
    SELECT
        p.*,

        CASE
            WHEN p.balance_before_raw < 0
                THEN 0.0
            ELSE p.balance_before_raw
        END AS balance_before

    FROM previous_balance p
),

principal_calc AS (
    SELECT
        i.*,

        i.balance_before * i.period_rate
            AS interest_payment,

        CASE
            WHEN i.n = i.n_payments - 1
                THEN i.balance_before

            WHEN i.annuity_payment
                 - i.balance_before * i.period_rate <= 0
                THEN 0.0

            WHEN i.annuity_payment
                 - i.balance_before * i.period_rate
                 >= i.balance_before
                THEN i.balance_before

            ELSE
                i.annuity_payment
                - i.balance_before * i.period_rate
        END AS principal_payment

    FROM interest_calc i
)

SELECT
    loan_id,
    method_id,
    is_floatrate,
    payment_date AS d,

    rate_int,
    rate_trf,

    CAST(principal_payment AS decimal(28,6))
        AS principal_payment,

    CAST(interest_payment AS decimal(28,6))
        AS interest_payment

INTO #payments

FROM principal_calc

OPTION (MAXDOP 2, RECOMPILE);

CREATE CLUSTERED INDEX IX_payments
ON #payments (method_id, is_floatrate, d);


/* ============================================================
   6. АГРЕГИРУЕМ ПЛАТЕЖИ ПО ДАТАМ И ГРУППАМ

   Сохраняем взвешенные числители и знаменатели,
   чтобы корректно пересчитывать ставки после амортизации.
   ============================================================ */

SELECT
    method_id,
    is_floatrate,
    d,

    SUM(principal_payment) AS principal_payment,
    SUM(interest_payment) AS interest_payment,

    SUM(
        CASE
            WHEN rate_int IS NOT NULL
                THEN principal_payment * rate_int
            ELSE 0
        END
    ) AS principal_int_num,

    SUM(
        CASE
            WHEN rate_int IS NOT NULL
                THEN principal_payment
            ELSE 0
        END
    ) AS principal_int_den,

    SUM(
        CASE
            WHEN rate_trf IS NOT NULL
                THEN principal_payment * rate_trf
            ELSE 0
        END
    ) AS principal_trf_num,

    SUM(
        CASE
            WHEN rate_trf IS NOT NULL
                THEN principal_payment
            ELSE 0
        END
    ) AS principal_trf_den

INTO #daily_payments

FROM #payments

GROUP BY
    method_id,
    is_floatrate,
    d;

CREATE UNIQUE CLUSTERED INDEX IX_daily_payments
ON #daily_payments (method_id, is_floatrate, d);


/* ============================================================
   7. ИСХОДНЫЙ ПОРТФЕЛЬ

   Те же средневзвешенные показатели, что были
   в первоначальном скрипте.
   ============================================================ */

SELECT
    is_floatrate,

    SUM(out_rub) AS init_out,

    SUM(
        CASE
            WHEN rate_int IS NOT NULL
                THEN out_rub * rate_int
            ELSE 0
        END
    ) AS init_int_num,

    SUM(
        CASE
            WHEN rate_int IS NOT NULL
                THEN out_rub
            ELSE 0
        END
    ) AS init_int_den,

    SUM(
        CASE
            WHEN rate_trf IS NOT NULL
                THEN out_rub * rate_trf
            ELSE 0
        END
    ) AS init_trf_num,

    SUM(
        CASE
            WHEN rate_trf IS NOT NULL
                THEN out_rub
            ELSE 0
        END
    ) AS init_trf_den

INTO #init

FROM #base

GROUP BY is_floatrate;

CREATE UNIQUE CLUSTERED INDEX IX_init
ON #init (is_floatrate);


/* ============================================================
   8. ПОДНЕВНОЙ КАЛЕНДАРЬ
   ============================================================ */

SELECT TOP (DATEDIFF(day, @dt_rep, @d_end) + 1)
    DATEADD(
        day,
        CAST(ROW_NUMBER() OVER (
            ORDER BY (SELECT NULL)
        ) - 1 AS int),
        DATEADD(day, -1, @dt_rep)
    ) AS d

INTO #calendar

FROM sys.all_objects a
CROSS JOIN sys.all_objects b;

CREATE UNIQUE CLUSTERED INDEX IX_calendar
ON #calendar (d);


/* ============================================================
   9. ПОДНЕВНОЙ ОСЗ И СТАВКИ

   ОСЗ = исходный ОСЗ - все погашения тела до даты.

   Проценты выплачиваются только в даты платежей.
   На остальных датах interest_payment = 0.

   Ставки пересчитываются по оставшемуся ОСЗ.
   ============================================================ */

;WITH grid AS (
    SELECT
        m.method_id,
        i.is_floatrate,
        c.d,

        i.init_out,
        i.init_int_num,
        i.init_int_den,
        i.init_trf_num,
        i.init_trf_den,

        ISNULL(p.principal_payment, 0)
            AS principal_payment,

        ISNULL(p.interest_payment, 0)
            AS interest_payment,

        ISNULL(p.principal_int_num, 0)
            AS principal_int_num,

        ISNULL(p.principal_int_den, 0)
            AS principal_int_den,

        ISNULL(p.principal_trf_num, 0)
            AS principal_trf_num,

        ISNULL(p.principal_trf_den, 0)
            AS principal_trf_den

    FROM #init i

    CROSS JOIN (
        SELECT 1 AS method_id
        UNION ALL
        SELECT 2
    ) m

    CROSS JOIN #calendar c

    LEFT JOIN #daily_payments p
        ON p.method_id = m.method_id
       AND p.is_floatrate = i.is_floatrate
       AND p.d = c.d
),

cumulative AS (
    SELECT
        g.*,

        SUM(principal_payment) OVER (
            PARTITION BY method_id, is_floatrate
            ORDER BY d
            ROWS UNBOUNDED PRECEDING
        ) AS cum_principal,

        SUM(principal_int_num) OVER (
            PARTITION BY method_id, is_floatrate
            ORDER BY d
            ROWS UNBOUNDED PRECEDING
        ) AS cum_int_num,

        SUM(principal_int_den) OVER (
            PARTITION BY method_id, is_floatrate
            ORDER BY d
            ROWS UNBOUNDED PRECEDING
        ) AS cum_int_den,

        SUM(principal_trf_num) OVER (
            PARTITION BY method_id, is_floatrate
            ORDER BY d
            ROWS UNBOUNDED PRECEDING
        ) AS cum_trf_num,

        SUM(principal_trf_den) OVER (
            PARTITION BY method_id, is_floatrate
            ORDER BY d
            ROWS UNBOUNDED PRECEDING
        ) AS cum_trf_den

    FROM grid g
)

SELECT
    method_id,
    d AS [date],
    is_floatrate,

    CAST(
        CASE
            WHEN init_out - cum_principal < 0
             AND init_out - cum_principal > -0.01
                THEN 0
            ELSE init_out - cum_principal
        END
        AS decimal(28,2)
    ) AS out_rub,

    CAST(
        (init_int_num - cum_int_num)
        / NULLIF(init_int_den - cum_int_den, 0)
        AS decimal(12,6)
    ) AS rate_int_srvz,

    CAST(
        (init_trf_num - cum_trf_num)
        / NULLIF(init_trf_den - cum_trf_den, 0)
        AS decimal(12,6)
    ) AS rate_trf_srvz,

    CAST(interest_payment AS decimal(28,2))
        AS interest_payment,

    CAST(principal_payment AS decimal(28,2))
        AS principal_payment,

    CAST(
        interest_payment + principal_payment
        AS decimal(28,2)
    ) AS total_payment

INTO #daily

FROM cumulative

OPTION (RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_daily
ON #daily (method_id, [date], is_floatrate);


/* ============================================================
   РЕЗУЛЬТАТ 1.

   АННУИТЕТ ОТ ФАКТИЧЕСКОГО ОСЗ НА ДАТУ СРЕЗА

   Первый процентный период начинается с @dt_rep.
   ============================================================ */

SELECT
    [date],
    is_floatrate,
    is_floatrate AS rate_type,

    out_rub,
    rate_int_srvz,
    rate_trf_srvz,

    interest_payment,
    principal_payment,
    total_payment

FROM #daily

WHERE method_id = 1

ORDER BY
    [date],
    is_floatrate;


/* ============================================================
   РЕЗУЛЬТАТ 2.

   АННУИТЕТ С УЧЕТОМ ДАТЫ ОТКРЫТИЯ

   Первый процентный период отсчитывается от
   предыдущего конца месяца / DT_OPEN_FACT.

   Исходный ОСЗ всё равно фактический на @dt_rep.
   ============================================================ */

SELECT
    [date],
    is_floatrate,
    is_floatrate AS rate_type,

    out_rub,
    rate_int_srvz,
    rate_trf_srvz,

    interest_payment,
    principal_payment,
    total_payment

FROM #daily

WHERE method_id = 2

ORDER BY
    [date],
    is_floatrate;
