USE [ALM];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2025-01-01';
DECLARE @DateTo   date = CAST(GETDATE() AS date);  -- "Н.В."

;WITH base AS (
    SELECT
        t.dt_rep,
        t.out_rub,
        t.PROD_NAME_res,
        t.TSEGMENTNAME
    FROM [ALM].[ALM].[balance_rest_all] t WITH (NOLOCK)
    WHERE
        t.dt_rep BETWEEN @DateFrom AND @DateTo
        AND t.section_name = N'Срочные'
        AND t.block_name   = N'Привлечение ФЛ'
        AND t.od_flag      = 1
        AND t.cur          = '810'
        AND t.out_rub      IS NOT NULL
        AND t.out_rub     >= 0
),
labeled AS (
    SELECT
        dt_rep,
        CASE
            WHEN PROD_NAME_res IN (N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
                                   N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2',
                                   N'Надёжный Мегафон', N'Надёжный процент')
                THEN N'промо ФУ'
            WHEN PROD_NAME_res IN (N'Надёжный')
                THEN N'ФУ'
            WHEN PROD_NAME_res IN (N'ДОМа надёжно')
                THEN N'Банки.ру'
            WHEN PROD_NAME_res IN (N'Всё в ДОМ')
                THEN N'Сравни.ру'
            ELSE
                CASE
                    WHEN TSEGMENTNAME IN (N'ДЧБО')
                        THEN N'Стандартная линейка УЧК'
                    ELSE N'Стандартная линейка Розница'
                END
        END AS category,
        out_rub
    FROM base
)
SELECT
    dt_rep,
    category,
    SUM(out_rub) AS sum_out_rub
FROM labeled
GROUP BY
    dt_rep, category
ORDER BY
    dt_rep, category;
