USE [ALM];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2026-05-01';
DECLARE @DateTo   date = '2026-05-05';

;WITH base AS (
    SELECT
        t.dt_rep,
        t.section_name,
        t.TSEGMENTNAME,
        t.PROD_NAME_res,
        CAST(t.out_rub AS decimal(20,2)) AS out_rub
    FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
    WHERE
        t.dt_rep BETWEEN @DateFrom AND @DateTo
        AND t.section_name IN (
            N'Срочные',
            N'Накопительный счёт'
        )
        AND t.block_name = N'Привлечение ФЛ'
        AND t.od_flag = 1
        AND t.cur = '810'
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
        AND (
            t.section_name <> N'Накопительный счёт'
            OR t.acc_role = N'LIAB'
        )
),

labeled AS (
    SELECT
        dt_rep,

        CASE

            /* ---------------- НАКОПИТЕЛЬНЫЕ СЧЕТА ---------------- */

            WHEN section_name = N'Накопительный счёт'
                 AND TSEGMENTNAME = N'ДЧБО'
                THEN N'НС УЧК'

            WHEN section_name = N'Накопительный счёт'
                THEN N'НС Розница'


            /* ---------------- СРОЧНЫЕ ---------------- */

            WHEN section_name = N'Срочные'
                 AND PROD_NAME_res = N'ДОМа надёжно'
                THEN N'Банки.ру'

            WHEN section_name = N'Срочные'
                 AND PROD_NAME_res = N'Всё в ДОМ'
                THEN N'Сравни.ру'

            WHEN section_name = N'Срочные'
                 AND PROD_NAME_res IN (
                    N'Надёжный прайм',
                    N'НадёжныйVIP',
                    N'Надёжный премиум',
                    N'Надёжный промо',
                    N'Надёжный старт',
                    N'Надёжный Т2',
                    N'Надёжный Мегафон',
                    N'Надёжный процент',
                    N'Могучий'
                 )
                THEN N'Промо ФУ'

            WHEN section_name = N'Срочные'
                 AND PROD_NAME_res = N'Надёжный'
                THEN N'ФУ'

            WHEN section_name = N'Срочные'
                 AND TSEGMENTNAME = N'ДЧБО'
                THEN N'Стандартная линейка УЧК'

            WHEN section_name = N'Срочные'
                THEN N'Стандартная линейка Розница'

        END AS category,

        out_rub

    FROM base
)

SELECT
    dt_rep,
    category,
    SUM(out_rub) AS sum_out_rub
FROM labeled
WHERE category IS NOT NULL
GROUP BY
    dt_rep,
    category
ORDER BY
    dt_rep,
    CASE category
        WHEN N'Банки.ру'                    THEN 1
        WHEN N'Сравни.ру'                   THEN 2
        WHEN N'Промо ФУ'                    THEN 3
        WHEN N'ФУ'                          THEN 4
        WHEN N'Стандартная линейка УЧК'     THEN 5
        WHEN N'Стандартная линейка Розница' THEN 6
        WHEN N'НС УЧК'                      THEN 7
        WHEN N'НС Розница'                  THEN 8
    END;
