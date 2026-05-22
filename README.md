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

IF OBJECT_ID('tempdb..#portfolio_agg') IS NOT NULL DROP TABLE #portfolio_agg;
IF OBJECT_ID('tempdb..#portfolio_4m_agg') IS NOT NULL DROP TABLE #portfolio_4m_agg;
IF OBJECT_ID('tempdb..#opened_agg') IS NOT NULL DROP TABLE #opened_agg;
IF OBJECT_ID('tempdb..#opened_4m_agg') IS NOT NULL DROP TABLE #opened_4m_agg;

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
-- 3. Портфель на конец месяца: поклиентное полотно
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
-- 5. Открытые в этом месяце вклады: поклиентное полотно
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
-- 7. Агрегация каждой категории до одной строки на месяц
------------------------------------------------------------

SELECT
      dt_rep
    , COUNT(DISTINCT cli_id) AS client_cnt
    , SUM(con_cnt)           AS con_cnt
    , SUM(out_rub)           AS out_rub
    , CAST(SUM(con_cnt) * 1.0 / NULLIF(COUNT(DISTINCT cli_id), 0) AS decimal(18,4))
        AS avg_con_cnt_per_client
    , CAST(SUM(out_rub) * 1.0 / NULLIF(COUNT(DISTINCT cli_id), 0) AS decimal(18,2))
        AS avg_out_rub_per_client
INTO #portfolio_agg
FROM #portfolio_client
GROUP BY dt_rep;


SELECT
      dt_rep
    , COUNT(DISTINCT cli_id) AS client_cnt
    , SUM(con_cnt)           AS con_cnt
    , SUM(out_rub)           AS out_rub
    , CAST(SUM(con_cnt) * 1.0 / NULLIF(COUNT(DISTINCT cli_id), 0) AS decimal(18,4))
        AS avg_con_cnt_per_client
    , CAST(SUM(out_rub) * 1.0 / NULLIF(COUNT(DISTINCT cli_id), 0) AS decimal(18,2))
        AS avg_out_rub_per_client
INTO #portfolio_4m_agg
FROM #portfolio_client_4m
GROUP BY dt_rep;


SELECT
      dt_rep
    , COUNT(DISTINCT cli_id) AS client_cnt
    , SUM(con_cnt)           AS con_cnt
    , SUM(out_rub)           AS out_rub
    , CAST(SUM(con_cnt) * 1.0 / NULLIF(COUNT(DISTINCT cli_id), 0) AS decimal(18,4))
        AS avg_con_cnt_per_client
    , CAST(SUM(out_rub) * 1.0 / NULLIF(COUNT(DISTINCT cli_id), 0) AS decimal(18,2))
        AS avg_out_rub_per_client
INTO #opened_agg
FROM #opened_client
GROUP BY dt_rep;


SELECT
      dt_rep
    , COUNT(DISTINCT cli_id) AS client_cnt
    , SUM(con_cnt)           AS con_cnt
    , SUM(out_rub)           AS out_rub
    , CAST(SUM(con_cnt) * 1.0 / NULLIF(COUNT(DISTINCT cli_id), 0) AS decimal(18,4))
        AS avg_con_cnt_per_client
    , CAST(SUM(out_rub) * 1.0 / NULLIF(COUNT(DISTINCT cli_id), 0) AS decimal(18,2))
        AS avg_out_rub_per_client
INTO #opened_4m_agg
FROM #opened_client_4m
GROUP BY dt_rep;


------------------------------------------------------------
-- 8. Финальный отчет: одна строка на месяц
------------------------------------------------------------

SELECT
      m.dt_rep

    --------------------------------------------------------
    -- Портфель на конец месяца
    --------------------------------------------------------
    , ISNULL(pa.client_cnt, 0) AS portfolio_client_cnt
    , ISNULL(pa.con_cnt, 0)    AS portfolio_con_cnt
    , ISNULL(pa.out_rub, 0)    AS portfolio_out_rub
    , pa.avg_con_cnt_per_client AS portfolio_avg_con_cnt_per_client
    , pa.avg_out_rub_per_client AS portfolio_avg_out_rub_per_client


    --------------------------------------------------------
    -- Портфель на конец месяца, клиенты с объемом >= 4 млн
    --------------------------------------------------------
    , ISNULL(p4.client_cnt, 0) AS portfolio_4m_client_cnt
    , ISNULL(p4.con_cnt, 0)    AS portfolio_4m_con_cnt
    , ISNULL(p4.out_rub, 0)    AS portfolio_4m_out_rub
    , p4.avg_con_cnt_per_client AS portfolio_4m_avg_con_cnt_per_client
    , p4.avg_out_rub_per_client AS portfolio_4m_avg_out_rub_per_client


    --------------------------------------------------------
    -- Открытые в этом месяце
    --------------------------------------------------------
    , ISNULL(oa.client_cnt, 0) AS opened_client_cnt
    , ISNULL(oa.con_cnt, 0)    AS opened_con_cnt
    , ISNULL(oa.out_rub, 0)    AS opened_out_rub
    , oa.avg_con_cnt_per_client AS opened_avg_con_cnt_per_client
    , oa.avg_out_rub_per_client AS opened_avg_out_rub_per_client


    --------------------------------------------------------
    -- Открытые в этом месяце, клиенты с объемом >= 4 млн
    --------------------------------------------------------
    , ISNULL(o4.client_cnt, 0) AS opened_4m_client_cnt
    , ISNULL(o4.con_cnt, 0)    AS opened_4m_con_cnt
    , ISNULL(o4.out_rub, 0)    AS opened_4m_out_rub
    , o4.avg_con_cnt_per_client AS opened_4m_avg_con_cnt_per_client
    , o4.avg_out_rub_per_client AS opened_4m_avg_out_rub_per_client

FROM #months m
LEFT JOIN #portfolio_agg pa
    ON m.dt_rep = pa.dt_rep
LEFT JOIN #portfolio_4m_agg p4
    ON m.dt_rep = p4.dt_rep
LEFT JOIN #opened_agg oa
    ON m.dt_rep = oa.dt_rep
LEFT JOIN #opened_4m_agg o4
    ON m.dt_rep = o4.dt_rep
ORDER BY
    m.dt_rep;
