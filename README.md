USE [ALM];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2024-01-01';
DECLARE @DateTo   date = '2025-10-31';
DECLARE @Today    date = GETDATE();
DECLARE @LastDay  date = DATEADD(DAY, -2, @Today);

;WITH month_ends AS (
    -- Все месяцы от начальной до конечной даты
    SELECT DATEADD(MONTH, n, EOMONTH(@DateFrom, -1)) AS month_start,
           EOMONTH(DATEADD(MONTH, n, @DateFrom))      AS month_end
    FROM master..spt_values
    WHERE type = 'P' AND DATEADD(MONTH, n, @DateFrom) <= @DateTo
),
effective_dates AS (
    SELECT
        CASE
            WHEN month_end < @LastDay THEN month_end    -- полный месяц
            ELSE (
                SELECT MAX(dt_rep)
                FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
                WHERE t.dt_rep <= @LastDay
                  AND t.section_name = N'Срочные'
                  AND t.block_name   = N'Привлечение ФЛ'
                  AND t.od_flag      = 1
                  AND t.cur          = '810'
                  AND t.AP           = N'Пассив'
                  AND t.acc_role     = N'LIAB'
                  AND t.tprod_name   = N'Вклады ФЛ'
            )
        END AS dt_rep
    FROM month_ends
),
base AS (
    SELECT
        t.dt_rep,
        t.cli_id,
        t.out_rub,
        t.PROD_NAME_res
    FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
    WHERE
        t.dt_rep IN (SELECT dt_rep FROM effective_dates)
        AND t.section_name = N'Срочные'
        AND t.block_name   = N'Привлечение ФЛ'
        AND t.od_flag      = 1
        AND t.cur          = '810'
        AND t.out_rub      > 0
        AND t.AP           = N'Пассив'
        AND t.acc_role     = N'LIAB'
        AND t.tprod_name   = N'Вклады ФЛ'
),
labeled AS (
    SELECT
        dt_rep,
        cli_id,
        CASE
            WHEN PROD_NAME_res IN (
                N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
                N'Надёжный промо', N'Надёжный старт',
                N'Надёжный T2', N'Надёжный Мегафон',
                N'Надёжный прайм', N'Надёжный процент'
            ) THEN N'ФУ'
            WHEN PROD_NAME_res = N'ДОМа надёжно'
                 THEN N'Банки.ру'
            ELSE NULL
        END AS category,
        out_rub
    FROM base
    WHERE PROD_NAME_res IN (
        N'ДОМа надёжно',
        N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
        N'Надёжный промо', N'Надёжный старт',
        N'Надёжный T2', N'Надёжный Мегафон',
        N'Надёжный прайм', N'Надёжный процент'
    )
)
SELECT
    l.dt_rep,
    l.category,
    SUM(l.out_rub)           AS sum_out_rub,
    COUNT(DISTINCT l.cli_id) AS cnt_cli
FROM labeled l
GROUP BY
    l.dt_rep, l.category
ORDER BY
    l.dt_rep, l.category
OPTION (MAXRECURSION 0);
