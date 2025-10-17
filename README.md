USE [ALM];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2024-01-01';
DECLARE @DateTo   date = '2025-10-31';
DECLARE @Today    date = GETDATE();
DECLARE @LastDay  date = DATEADD(DAY, -2, @Today);

-- =============================================
-- 1️⃣ Построение календаря нужных dt_rep
-- =============================================
IF OBJECT_ID('tempdb..#month_dates') IS NOT NULL DROP TABLE #month_dates;

;WITH months AS (
    SELECT CAST(@DateFrom AS date) AS month_start
    UNION ALL
    SELECT DATEADD(MONTH, 1, month_start)
    FROM months
    WHERE DATEADD(MONTH, 1, month_start) <= @DateTo
)
SELECT
    CASE
        WHEN EOMONTH(month_start) <= @LastDay
             THEN EOMONTH(month_start)
        ELSE @LastDay
    END AS dt_rep
INTO #month_dates
FROM months
OPTION (MAXRECURSION 0);

-- =============================================
-- 2️⃣ Основная выгрузка: только за нужные даты
-- =============================================
SELECT
    t.dt_rep,
    CASE
        WHEN t.PROD_NAME_res IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный T2', N'Надёжный Мегафон',
            N'Надёжный прайм', N'Надёжный процент'
        ) THEN N'ФУ'
        WHEN t.PROD_NAME_res = N'ДОМа надёжно' THEN N'Банки.ру'
        ELSE NULL
    END AS category,
    SUM(t.out_rub)           AS sum_out_rub,
    COUNT(DISTINCT t.cli_id) AS cnt_cli
FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
WHERE
    t.dt_rep IN (SELECT dt_rep FROM #month_dates)
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag      = 1
    AND t.cur          = '810'
    AND t.out_rub      > 0
    AND t.AP           = N'Пассив'
    AND t.acc_role     = N'LIAB'
    AND t.tprod_name   = N'Вклады ФЛ'
    AND t.PROD_NAME_res IN (
        N'ДОМа надёжно',
        N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
        N'Надёжный промо', N'Надёжный старт',
        N'Надёжный T2', N'Надёжный Мегафон',
        N'Надёжный прайм', N'Надёжный процент'
    )
GROUP BY
    t.dt_rep,
    CASE
        WHEN t.PROD_NAME_res IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный T2', N'Надёжный Мегафон',
            N'Надёжный прайм', N'Надёжный процент'
        ) THEN N'ФУ'
        WHEN t.PROD_NAME_res = N'ДОМа надёжно' THEN N'Банки.ру'
        ELSE NULL
    END
ORDER BY
    t.dt_rep,
    category;
