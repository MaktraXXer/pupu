USE [ALM];
SET NOCOUNT ON;

DECLARE @dt_prev date = '2026-05-25';
DECLARE @dt_curr date = '2026-05-26';

DROP TABLE IF EXISTS #active_clients_25;
DROP TABLE IF EXISTS #clients_with_nadezhny_25;
DROP TABLE IF EXISTS #clients_with_nadezhny_26;
DROP TABLE IF EXISTS #target_clients;


-- 1. Все действующие клиенты банка на 25 мая
SELECT DISTINCT
    t.cli_id
INTO #active_clients_25
FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
WHERE
    t.dt_rep = @dt_prev
    AND t.section_name IN (N'Срочные', N'До востребования', N'Накопительный счёт')
    AND t.block_name = N'Привлечение ФЛ'
    AND t.od_flag = 1
    -- AND t.cur = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;


CREATE CLUSTERED INDEX IX_active_clients_25
ON #active_clients_25 (cli_id);


-- 2. Клиенты, у кого на 25 мая уже был Надёжный
SELECT DISTINCT
    t.cli_id
INTO #clients_with_nadezhny_25
FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
WHERE
    t.dt_rep = @dt_prev
    AND t.section_name = N'Срочные'
    AND t.block_name = N'Привлечение ФЛ'
    AND t.od_flag = 1
    -- AND t.cur = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
    AND t.PROD_NAME_res = N'Надёжный';


CREATE CLUSTERED INDEX IX_clients_with_nadezhny_25
ON #clients_with_nadezhny_25 (cli_id);


-- 3. Клиенты, у кого на 26 мая появился/есть Надёжный
SELECT DISTINCT
    t.cli_id
INTO #clients_with_nadezhny_26
FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
WHERE
    t.dt_rep = @dt_curr
    AND t.section_name = N'Срочные'
    AND t.block_name = N'Привлечение ФЛ'
    AND t.od_flag = 1
    -- AND t.cur = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
    AND t.PROD_NAME_res = N'Надёжный';


CREATE CLUSTERED INDEX IX_clients_with_nadezhny_26
ON #clients_with_nadezhny_26 (cli_id);


-- 4. Целевые клиенты:
-- были в банке на 25 мая,
-- не имели Надёжный на 25 мая,
-- имеют Надёжный на 26 мая
SELECT
    a.cli_id
INTO #target_clients
FROM #active_clients_25 a
INNER JOIN #clients_with_nadezhny_26 n26
    ON n26.cli_id = a.cli_id
LEFT JOIN #clients_with_nadezhny_25 n25
    ON n25.cli_id = a.cli_id
WHERE
    n25.cli_id IS NULL;


CREATE CLUSTERED INDEX IX_target_clients
ON #target_clients (cli_id);


-- 5. Один селект: баланс этих клиентов на 25 и 26 мая
SELECT
    t.dt_rep,
    t.cli_id,
    t.con_id,
    t.section_name,
    t.block_name,
    t.PROD_NAME_res,
    t.out_rub,
    t.rate_con,
    t.dt_open,
    t.dt_close_plan,

    CASE
        WHEN t.PROD_NAME_res = N'Надёжный'
            THEN 1
        ELSE 0
    END AS is_nadezhny,

    CASE
        WHEN t.dt_rep = @dt_prev
            THEN N'25.05 до появления Надёжного'
        WHEN t.dt_rep = @dt_curr
            THEN N'26.05 после появления Надёжного'
    END AS period_flag

FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
INNER JOIN #target_clients c
    ON c.cli_id = t.cli_id
WHERE
    t.dt_rep IN (@dt_prev, @dt_curr)
    AND t.section_name IN (N'Срочные', N'До востребования', N'Накопительный счёт')
    AND t.block_name = N'Привлечение ФЛ'
    AND t.od_flag = 1
    -- AND t.cur = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
ORDER BY
    t.cli_id,
    t.dt_rep,
    t.section_name,
    t.PROD_NAME_res,
    t.con_id;
