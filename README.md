Да. Логика такая:

сначала на том же срезе @base_date определяем клиента на уровне cli_id:
если у клиента есть хотя бы одна строка с TSEGMENTNAME = 'ДЧБО', то весь cli_id считаем ДЧБО.
Иначе весь cli_id считаем розницей.

Ниже два отдельных запроса.

Запрос 1. Только клиенты ДЧБО

DECLARE @base_date DATE = CAST(DATEADD(DAY, -2, GETDATE()) AS DATE);

WITH ranges AS (
    SELECT '0-1,5' AS range_name, -100.0 AS lower_bound, 1.5 AS upper_bound
    UNION ALL SELECT '1,5-5', 1.5, 5.0
    UNION ALL SELECT '5-10', 5.0, 10.0
    UNION ALL SELECT '10-100', 10.0, 100.0
    UNION ALL SELECT '100-500', 100.0, 500.0
    UNION ALL SELECT '500-1000', 500.0, 1000.0
    UNION ALL SELECT '1000-5000', 1000.0, 5000.0
    UNION ALL SELECT '5000-15000', 5000.0, 15000.0
    UNION ALL SELECT '>15000', 15000.0, 10000000000.0
),
base_rows AS (
    SELECT
        cli_id,
        out_rub,
        TSEGMENTNAME
    FROM [ALM].[balance_rest_all] WITH (NOLOCK)
    WHERE dt_rep = @base_date
      AND od_flag = 1
      AND block_name = 'Привлечение ФЛ'
      AND section_name NOT IN (
            'Аккредитивы',
            'Брокерское обслуживание',
            'Аккредитив под строительство'
      )
),
client_types AS (
    SELECT
        cli_id,
        CASE
            WHEN MAX(CASE WHEN TSEGMENTNAME = 'ДЧБО' THEN 1 ELSE 0 END) = 1
                THEN N'ДЧБО'
            ELSE N'Розница'
        END AS client_type
    FROM base_rows
    GROUP BY cli_id
),
client_balances AS (
    SELECT
        b.cli_id,
        SUM(b.out_rub) / 1000000.0 AS total_balance
    FROM base_rows b
    INNER JOIN client_types ct
        ON b.cli_id = ct.cli_id
    WHERE ct.client_type = N'ДЧБО'
    GROUP BY b.cli_id
),
total_portfolio AS (
    SELECT SUM(total_balance) AS total_sum
    FROM client_balances
)
SELECT
    r.range_name,
    r.lower_bound,
    r.upper_bound,
    COUNT(c.cli_id) AS [Кол-во],
    COALESCE(SUM(c.total_balance), 0) AS [Сумма],
    COALESCE(SUM(c.total_balance) / NULLIF(t.total_sum, 0), 0) AS [В% от портфеля]
FROM ranges r
LEFT JOIN client_balances c
    ON c.total_balance > r.lower_bound
   AND c.total_balance <= r.upper_bound
CROSS JOIN total_portfolio t
GROUP BY
    r.range_name,
    r.lower_bound,
    r.upper_bound,
    t.total_sum
ORDER BY r.lower_bound;

Запрос 2. Только клиенты розницы

DECLARE @base_date DATE = CAST(DATEADD(DAY, -2, GETDATE()) AS DATE);

WITH ranges AS (
    SELECT '0-1,5' AS range_name, -100.0 AS lower_bound, 1.5 AS upper_bound
    UNION ALL SELECT '1,5-5', 1.5, 5.0
    UNION ALL SELECT '5-10', 5.0, 10.0
    UNION ALL SELECT '10-100', 10.0, 100.0
    UNION ALL SELECT '100-500', 100.0, 500.0
    UNION ALL SELECT '500-1000', 500.0, 1000.0
    UNION ALL SELECT '1000-5000', 1000.0, 5000.0
    UNION ALL SELECT '5000-15000', 5000.0, 15000.0
    UNION ALL SELECT '>15000', 15000.0, 10000000000.0
),
base_rows AS (
    SELECT
        cli_id,
        out_rub,
        TSEGMENTNAME
    FROM [ALM].[balance_rest_all] WITH (NOLOCK)
    WHERE dt_rep = @base_date
      AND od_flag = 1
      AND block_name = 'Привлечение ФЛ'
      AND section_name NOT IN (
            'Аккредитивы',
            'Брокерское обслуживание',
            'Аккредитив под строительство'
      )
),
client_types AS (
    SELECT
        cli_id,
        CASE
            WHEN MAX(CASE WHEN TSEGMENTNAME = 'ДЧБО' THEN 1 ELSE 0 END) = 1
                THEN N'ДЧБО'
            ELSE N'Розница'
        END AS client_type
    FROM base_rows
    GROUP BY cli_id
),
client_balances AS (
    SELECT
        b.cli_id,
        SUM(b.out_rub) / 1000000.0 AS total_balance
    FROM base_rows b
    INNER JOIN client_types ct
        ON b.cli_id = ct.cli_id
    WHERE ct.client_type = N'Розница'
    GROUP BY b.cli_id
),
total_portfolio AS (
    SELECT SUM(total_balance) AS total_sum
    FROM client_balances
)
SELECT
    r.range_name,
    r.lower_bound,
    r.upper_bound,
    COUNT(c.cli_id) AS [Кол-во],
    COALESCE(SUM(c.total_balance), 0) AS [Сумма],
    COALESCE(SUM(c.total_balance) / NULLIF(t.total_sum, 0), 0) AS [В% от портфеля]
FROM ranges r
LEFT JOIN client_balances c
    ON c.total_balance > r.lower_bound
   AND c.total_balance <= r.upper_bound
CROSS JOIN total_portfolio t
GROUP BY
    r.range_name,
    r.lower_bound,
    r.upper_bound,
    t.total_sum
ORDER BY r.lower_bound;

Если хочешь, могу еще дать третью версию: один запрос, где сразу в выходе будет столбец client_type и две выборки будут в одном результате.
