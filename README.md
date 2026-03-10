/* ===================== ПАРАМЕТРЫ ===================== */
DECLARE @dt_to_mar date = '2026-03-05';
DECLARE @dt_to_feb date = '2026-02-05';
DECLARE @dt_to_oct date = '2025-10-05';

/* ===================== БАЗА ПЕРИОДОВ ===================== */
WITH periods AS (
    SELECT CAST('2026-03-01' AS date) AS dt_from, @dt_to_mar AS dt_to, '01.03-' + CONVERT(varchar(5), @dt_to_mar, 104) AS r_w
    UNION ALL
    SELECT CAST('2026-02-01' AS date), @dt_to_feb, '01.02-' + CONVERT(varchar(5), @dt_to_feb, 104)
    UNION ALL
    SELECT CAST('2025-10-01' AS date), @dt_to_oct, '01.10-' + CONVERT(varchar(5), @dt_to_oct, 104)
),
base AS (
    SELECT
        p.r_w,
        t.dt_rep,
        t.bank_name_main,
        t.spec_cat,
        t.transaction_type,
        t.direction_type,
        t.transit_flag,
        t.cli_biz,
        t.[целевая категория],
        t.[распознано_НС] + t.[распознано_СР] + t.[Распознано_ДВС] AS amt
    FROM ALM.[ehd].[VW_transfers_FL_AGG_tab] t WITH (NOLOCK)
    INNER JOIN periods p
        ON t.dt_rep BETWEEN p.dt_from AND p.dt_to
    WHERE 1=1
        AND t.direction_type <> 'Ошибка'
        AND t.[целевая категория] = 1
        AND t.transaction_type <> 'Внутренние переводы'
        AND t.transit_flag = 'Не транзит'
        AND t.cli_biz IN ('Розница', 'ДЧБО')
),
else_table_by_bank AS (
    SELECT
        r_w,
        bank_name_main AS bank,
        SUM(amt) AS saldo_else_bank
    FROM base
    WHERE spec_cat NOT IN ('BR', 'SR', 'FU')
      AND transaction_type <> 'Наличные'
    GROUP BY r_w, bank_name_main
    HAVING ABS(SUM(amt)) < POWER(10, 6) * 150
),
else_for_date AS (
    SELECT
        r_w,
        SUM(saldo_else_bank) AS saldo_else
    FROM else_table_by_bank
    GROUP BY r_w
),
big_banks AS (
    SELECT
        r_w,
        bank_name_main,
        SUM(amt) / POWER(10, 6) AS saldo
    FROM base
    WHERE spec_cat NOT IN ('BR', 'SR', 'FU')
      AND transaction_type <> 'Наличные'
    GROUP BY r_w, bank_name_main
    HAVING ABS(SUM(amt)) >= POWER(10, 6) * 150
),
market AS (
    SELECT
        r_w,
        'Market' AS bank_name_main,
        SUM(amt) / POWER(10, 6) AS saldo
    FROM base
    WHERE spec_cat IN ('BR', 'SR', 'FU')
      AND transaction_type <> 'Наличные'
    GROUP BY r_w
),
cash AS (
    SELECT
        r_w,
        'Наличные' AS bank_name_main,
        SUM(amt) / POWER(10, 6) AS saldo
    FROM base
    WHERE spec_cat NOT IN ('BR', 'SR', 'FU')
      AND transaction_type = 'Наличные'
    GROUP BY r_w
)

SELECT
    r_w,
    'else' AS bank_name_main,
    saldo_else / POWER(10, 6) AS saldo
FROM else_for_date

UNION ALL

SELECT
    r_w,
    bank_name_main,
    saldo
FROM big_banks

UNION ALL

SELECT
    r_w,
    bank_name_main,
    saldo
FROM market

UNION ALL

SELECT
    r_w,
    bank_name_main,
    saldo
FROM cash

ORDER BY
    r_w,
    bank_name_main;
