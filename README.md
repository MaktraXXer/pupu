USE [ALM];
SET NOCOUNT ON;

DECLARE @dt_rep date = '2026-06-23';

WITH market_products AS (
    SELECT prod_name_res
    FROM (VALUES
        (N'Надёжный прайм'),
        (N'Надёжный VIP'),
        (N'Надёжный премиум'),
        (N'Надёжный промо'),
        (N'Надёжный старт'),
        (N'Надёжный Т2'),
        (N'Надёжный'),
        (N'Надёжный Мегафон'),
        (N'Надёжный процент'),
        (N'Могучий')
    ) v(prod_name_res)
),
base AS (
    SELECT
        t.cli_id,
        t.section_name,
        t.prod_name_res,
        t.out_rub,
        CASE 
            WHEN mp.prod_name_res IS NOT NULL THEN 1 
            ELSE 0 
        END AS is_market_product
    FROM [ALM].[ALM].balance_rest_all t WITH (NOLOCK)
    LEFT JOIN market_products mp
        ON t.prod_name_res = mp.prod_name_res
    WHERE
        t.dt_rep      = @dt_rep
        AND t.block_name = N'Привлечение ФЛ'
        AND t.od_flag    = 1
        AND t.cur        = '810'
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
)
SELECT
    b.cli_id,

    SUM(CASE 
            WHEN b.section_name = N'Срочные'
             AND b.is_market_product = 1
            THEN b.out_rub 
            ELSE 0 
        END) AS summa_na_marketah,

    SUM(CASE 
            WHEN b.section_name = N'Срочные'
             AND b.is_market_product = 0
            THEN b.out_rub 
            ELSE 0 
        END) AS summa_na_vkladah,

    SUM(CASE 
            WHEN b.section_name = N'Накопительный счёт'
            THEN b.out_rub 
            ELSE 0 
        END) AS summa_na_ns,

    SUM(CASE 
            WHEN b.section_name = N'До востребования'
            THEN b.out_rub 
            ELSE 0 
        END) AS summa_na_dvs

FROM base b
GROUP BY b.cli_id
HAVING 
    SUM(CASE 
            WHEN b.section_name = N'Срочные'
             AND b.is_market_product = 1
            THEN b.out_rub 
            ELSE 0 
        END) > 0;
