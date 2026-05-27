USE [ALM];
SET NOCOUNT ON;

DECLARE @dt_prev date = '2026-05-25';
DECLARE @dt_curr date = '2026-05-26';

DROP TABLE IF EXISTS #target_clients;

SELECT DISTINCT
    t26.cli_id
INTO #target_clients
FROM [ALM].[ALM].[VW_balance_rest_all] t26 WITH (NOLOCK)
WHERE
    t26.dt_rep = @dt_curr
    AND t26.section_name = N'Срочные'
    AND t26.block_name = N'Привлечение ФЛ'
    AND t26.od_flag = 1
    -- AND t26.cur = '810'
    AND t26.out_rub IS NOT NULL
    AND t26.out_rub >= 0
    AND t26.PROD_NAME_res = N'Надёжный'
    AND NOT EXISTS (
        SELECT 1
        FROM [ALM].[ALM].[VW_balance_rest_all] t25 WITH (NOLOCK)
        WHERE
            t25.dt_rep = @dt_prev
            AND t25.cli_id = t26.cli_id
            AND t25.section_name IN (N'Срочные', N'До востребования', N'Накопительный счёт')
            AND t25.block_name = N'Привлечение ФЛ'
            AND t25.od_flag = 1
            -- AND t25.cur = '810'
            AND t25.out_rub IS NOT NULL
            AND t25.out_rub >= 0
            AND t25.PROD_NAME_res = N'Надёжный'
    );

CREATE CLUSTERED INDEX IX_target_clients
ON #target_clients (cli_id);


SELECT
    t.dt_rep,
    t.cli_id,
    t.con_id,
    t.section_name,
    t.PROD_NAME_res,
    t.dt_open,
    t.dt_close_plan,
    t.out_rub,
    t.rate_con,
    CASE
        WHEN t.PROD_NAME_res = N'Надёжный' THEN 1
        ELSE 0
    END AS is_target_product
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
