USE [ALM];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2025-06-01';
DECLARE @DateTo   date = '2026-04-30';

IF OBJECT_ID('tempdb..#months') IS NOT NULL DROP TABLE #months;
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
IF OBJECT_ID('tempdb..#portfolio_client') IS NOT NULL DROP TABLE #portfolio_client;
IF OBJECT_ID('tempdb..#portfolio_client_4m') IS NOT NULL DROP TABLE #portfolio_client_4m;
IF OBJECT_ID('tempdb..#opened_client') IS NOT NULL DROP TABLE #opened_client;
IF OBJECT_ID('tempdb..#opened_client_4m') IS NOT NULL DROP TABLE #opened_client_4m;

------------------------------------------------------------
-- 1. Календарь концов месяцев
------------------------------------------------------------

;WITH m AS (
    SELECT DATEFROMPARTS(YEAR(@DateFrom), MONTH(@DateFrom), 1) AS month_start

    UNION ALL

    SELECT DATEADD(month, 1, month_start)
    FROM m
    WHERE DATEADD(month, 1, month_start) <= DATEFROMPARTS(YEAR(@DateTo), MONTH(@DateTo), 1)
)
SELECT
      EOMONTH(month_start) AS dt_rep
    , month_start
    , EOMONTH(month_start) AS month_end
INTO #months
FROM m
OPTION (MAXRECURSION 100);


------------------------------------------------------------
-- 2. База по нужным вкладам на конец каждого месяца
------------------------------------------------------------

SELECT
      CAST(t.dt_rep AS date)  AS dt_rep
    , t.cli_id
    , t.con_id
    , CAST(t.dt_open AS date) AS dt_open
    , t.out_rub
INTO #base
FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
JOIN #months m
    ON CAST(t.dt_rep AS date) = m.dt_rep
WHERE
        t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag      = 1
    AND t.cur          = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
    AND t.PROD_NAME_res IN (N'Надёжный', N'Надёжный VIP');


------------------------------------------------------------
-- 3. Портфель на конец месяца: агрегация до клиента
------------------------------------------------------------

SELECT
      b.dt_rep
    , b.cli_id
    , COUNT(DISTINCT b.con_id) AS con_cnt
    , SUM(b.out_rub)           AS out_rub
INTO #portfolio_client
FROM #base b
GROUP BY
      b.dt_rep
    , b.cli_id;


------------------------------------------------------------
-- 4. Портфель на конец месяца: клиенты с объемом >= 4 млн
------------------------------------------------------------

SELECT
      pc.dt_rep
    , pc.cli_id
    , pc.con_cnt
    , pc.out_rub
INTO #portfolio_client_4m
FROM #portfolio_client pc
WHERE pc.out_rub >= 4000000;


------------------------------------------------------------
-- 5. Открытые в этом месяце вклады: агрегация до клиента
------------------------------------------------------------

SELECT
      b.dt_rep
    , b.cli_id
    , COUNT(DISTINCT b.con_id) AS con_cnt
    , SUM(b.out_rub)           AS out_rub
INTO #opened_client
FROM #base b
JOIN #months m
    ON b.dt_rep = m.dt_rep
WHERE
        b.dt_open >= m.month_start
    AND b.dt_open <= m.month_end
GROUP BY
      b.dt_rep
    , b.cli_id;


------------------------------------------------------------
-- 6. Открытые в этом месяце: клиенты с объемом >= 4 млн
------------------------------------------------------------

SELECT
      oc.dt_rep
    , oc.cli_id
    , oc.con_cnt
    , oc.out_rub
INTO #opened_client_4m
FROM #opened_client oc
WHERE oc.out_rub >= 4000000;


------------------------------------------------------------
-- 7. Финальный отчет по месяцам
------------------------------------------------------------

SELECT
      m.dt_rep

    --------------------------------------------------------
    -- Портфель на конец месяца
    --------------------------------------------------------
    , COUNT(DISTINCT pc.cli_id) AS portfolio_client_cnt
    , ISNULL(SUM(pc.con_cnt), 0) AS portfolio_con_cnt
    , ISNULL(SUM(pc.out_rub), 0) AS portfolio_out_rub

    , CAST(
        ISNULL(SUM(pc.con_cnt), 0) * 1.0 
        / NULLIF(COUNT(DISTINCT pc.cli_id), 0)
      AS decimal(18,4)) AS portfolio_avg_con_cnt_per_client

    , CAST(
        ISNULL(SUM(pc.out_rub), 0) * 1.0 
        / NULLIF(COUNT(DISTINCT pc.cli_id), 0)
      AS decimal(18,2)) AS portfolio_avg_out_rub_per_client


    --------------------------------------------------------
    -- Портфель на конец месяца, клиенты с объемом >= 4 млн
    --------------------------------------------------------
    , COUNT(DISTINCT pc4.cli_id) AS portfolio_4m_client_cnt
    , ISNULL(SUM(pc4.con_cnt), 0) AS portfolio_4m_con_cnt
    , ISNULL(SUM(pc4.out_rub), 0) AS portfolio_4m_out_rub

    , CAST(
        ISNULL(SUM(pc4.con_cnt), 0) * 1.0 
        / NULLIF(COUNT(DISTINCT pc4.cli_id), 0)
      AS decimal(18,4)) AS portfolio_4m_avg_con_cnt_per_client

    , CAST(
        ISNULL(SUM(pc4.out_rub), 0) * 1.0 
        / NULLIF(COUNT(DISTINCT pc4.cli_id), 0)
      AS decimal(18,2)) AS portfolio_4m_avg_out_rub_per_client


    --------------------------------------------------------
    -- Открытые в этом месяце
    --------------------------------------------------------
    , COUNT(DISTINCT oc.cli_id) AS opened_client_cnt
    , ISNULL(SUM(oc.con_cnt), 0) AS opened_con_cnt
    , ISNULL(SUM(oc.out_rub), 0) AS opened_out_rub

    , CAST(
        ISNULL(SUM(oc.con_cnt), 0) * 1.0 
        / NULLIF(COUNT(DISTINCT oc.cli_id), 0)
      AS decimal(18,4)) AS opened_avg_con_cnt_per_client

    , CAST(
        ISNULL(SUM(oc.out_rub), 0) * 1.0 
        / NULLIF(COUNT(DISTINCT oc.cli_id), 0)
      AS decimal(18,2)) AS opened_avg_out_rub_per_client


    --------------------------------------------------------
    -- Открытые в этом месяце, клиенты с объемом >= 4 млн
    --------------------------------------------------------
    , COUNT(DISTINCT oc4.cli_id) AS opened_4m_client_cnt
    , ISNULL(SUM(oc4.con_cnt), 0) AS opened_4m_con_cnt
    , ISNULL(SUM(oc4.out_rub), 0) AS opened_4m_out_rub

    , CAST(
        ISNULL(SUM(oc4.con_cnt), 0) * 1.0 
        / NULLIF(COUNT(DISTINCT oc4.cli_id), 0)
      AS decimal(18,4)) AS opened_4m_avg_con_cnt_per_client

    , CAST(
        ISNULL(SUM(oc4.out_rub), 0) * 1.0 
        / NULLIF(COUNT(DISTINCT oc4.cli_id), 0)
      AS decimal(18,2)) AS opened_4m_avg_out_rub_per_client

FROM #months m
LEFT JOIN #portfolio_client pc
    ON m.dt_rep = pc.dt_rep

LEFT JOIN #portfolio_client_4m pc4
    ON m.dt_rep = pc4.dt_rep

LEFT JOIN #opened_client oc
    ON m.dt_rep = oc.dt_rep

LEFT JOIN #opened_client_4m oc4
    ON m.dt_rep = oc4.dt_rep

GROUP BY
    m.dt_rep

ORDER BY
    m.dt_rep;
