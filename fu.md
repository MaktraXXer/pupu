USE [ALM];
SET NOCOUNT ON;

DECLARE @dt_prev date = '2026-05-25';
DECLARE @dt_curr date = '2026-05-26';

DROP TABLE IF EXISTS #prod_list;

CREATE TABLE #prod_list (
    prod_name_res nvarchar(255) NOT NULL PRIMARY KEY
);

INSERT INTO #prod_list (prod_name_res)
VALUES
    (N'Надёжный прайм'),
    (N'Надёжный VIP'),
    (N'Надёжный премиум'),
    (N'Надёжный промо'),
    (N'Надёжный старт'),
    (N'Надёжный Т2'),
    (N'Надёжный Мегафон'),
    (N'Надёжный процент'),
    (N'Надёжный'),
    (N'Могучий');


DROP TABLE IF EXISTS #clients_without_prod_25;

SELECT
    t.cli_id
INTO #clients_without_prod_25
FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
LEFT JOIN #prod_list p
    ON p.prod_name_res = t.PROD_NAME_res
WHERE
    t.dt_rep = @dt_prev
    AND t.section_name IN (N'Срочные', N'До востребования', N'Накопительный счёт')
    AND t.block_name = N'Привлечение ФЛ'
    AND t.od_flag = 1
    -- AND t.cur = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
GROUP BY
    t.cli_id
HAVING
    COUNT(DISTINCT CASE WHEN p.prod_name_res IS NOT NULL THEN t.con_id END) = 0;


CREATE CLUSTERED INDEX IX_clients_without_prod_25
ON #clients_without_prod_25 (cli_id);


SELECT
    t.*
FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
INNER JOIN #clients_without_prod_25 c
    ON c.cli_id = t.cli_id
INNER JOIN #prod_list p
    ON p.prod_name_res = t.PROD_NAME_res
WHERE
    t.dt_rep = @dt_curr
    AND t.section_name = N'Срочные'
    AND t.block_name = N'Привлечение ФЛ'
    AND t.od_flag = 1
    -- AND t.cur = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;
