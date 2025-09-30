DECLARE @dt_rep date = '2025-08-31';

/* ===== 1) Снимок на дату отчёта -> #base ===== */
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;

SELECT
    t.TSegmentname,
    CAST(t.dt_close AS date) AS dt_close_d,
    t.out_rub,
    t.rate_con,
    t.rate_trf
INTO #base
FROM alm.[ALM].[vw_balance_rest_all] AS t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ЮЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.AP           = N'Пассив'
  AND t.dt_close     > t.dt_rep
  AND t.out_rub IS NOT NULL
  AND t.tprod_name   = N'Депозиты ЮЛ'
  AND t.TSegmentname IN (N'Ипотека', N'Розничный бизнес');

-- FIX: без INCLUDE в кластеризованном индексе
CREATE CLUSTERED INDEX IX_base_seg_close ON #base (TSegmentname, dt_close_d);

/* ===== 2) Границы и календарь без рекурсии ===== */
DECLARE @d_end date;
SELECT @d_end = MAX(dt_close_d) FROM #base;
-- на случай пустого набора: держим хотя бы @dt_rep
SET @d_end = ISNULL(@d_end, @dt_rep);

IF OBJECT_ID('tempdb..#calendar') IS NOT NULL DROP TABLE #calendar;

;WITH N AS (
    SELECT TOP (DATEDIFF(DAY, @dt_rep, @d_end) + 1)
           ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS n
    FROM sys.all_objects a CROSS JOIN sys.all_objects b
)
SELECT DATEADD(DAY, n, @dt_rep) AS d
INTO #calendar
FROM N;  -- FIX: добавлен FROM N

CREATE CLUSTERED INDEX IX_calendar_d ON #calendar(d);

/* ===== 3) Сегменты и сетка дат ===== */
IF OBJECT_ID('tempdb..#segments') IS NOT NULL DROP TABLE #segments;
SELECT DISTINCT TSegmentname INTO #segments FROM #base;

IF OBJECT_ID('tempdb..#grid') IS NOT NULL DROP TABLE #grid;
SELECT s.TSegmentname, c.d
INTO #grid
FROM #segments s
CROSS JOIN #calendar c;
CREATE CLUSTERED INDEX IX_grid_seg_d ON #grid(TSegmentname, d);

/* ===== 4) Закрытия по датам и их кумулятивные суммы ===== */
IF OBJECT_ID('tempdb..#closings') IS NOT NULL DROP TABLE #closings;

SELECT
    b.dt_close_d AS d,
    b.TSegmentname,
    SUM(b.out_rub)                                                    AS out_rub_close,
    SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END) AS trf_num,
    SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub END)              AS trf_den,
    SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END) AS con_num,
    SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END)              AS con_den
INTO #closings
FROM #base b
GROUP BY b.dt_close_d, b.TSegmentname;

CREATE CLUSTERED INDEX IX_closings_seg_d ON #closings(TSegmentname, d);

IF OBJECT_ID('tempdb..#closings_cum') IS NOT NULL DROP TABLE #closings_cum;

SELECT
    c.TSegmentname,
    c.d,
    SUM(c.out_rub_close) OVER (PARTITION BY c.TSegmentname ORDER BY c.d ROWS UNBOUNDED PRECEDING) AS cum_out_rub,
    SUM(c.trf_num)       OVER (PARTITION BY c.TSegmentname ORDER BY c.d ROWS UNBOUNDED PRECEDING) AS cum_trf_num,
    SUM(c.trf_den)       OVER (PARTITION BY c.TSegmentname ORDER BY c.d ROWS UNBOUNDED PRECEDING) AS cum_trf_den,
    SUM(c.con_num)       OVER (PARTITION BY c.TSegmentname ORDER BY c.d ROWS UNBOUNDED PRECEDING) AS cum_con_num,
    SUM(c.con_den)       OVER (PARTITION BY c.TSegmentname ORDER BY c.d ROWS UNBOUNDED PRECEDING) AS cum_con_den
INTO #closings_cum
FROM #closings c;

CREATE CLUSTERED INDEX IX_closings_cum_seg_d ON #closings_cum(TSegmentname, d);

/* ===== 5) Начальные итоги на @dt_rep (вес. средние по non-NULL ставкам) ===== */
IF OBJECT_ID('tempdb..#init') IS NOT NULL DROP TABLE #init;

SELECT
    b.TSegmentname,
    SUM(b.out_rub) AS init_out,
    SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END) AS init_trf_num,
    SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub END)               AS init_trf_den,
    SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END)  AS init_con_num,
    SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END)               AS init_con_den
INTO #init
FROM #base b
GROUP BY b.TSegmentname;

CREATE UNIQUE CLUSTERED INDEX IX_init_seg ON #init(TSegmentname);

/* ===== 6a) РЕЗУЛЬТАТ 1: Амортизация (включая 2025-08-31) ===== */
SELECT
    g.d                                 AS [date],
    g.TSegmentname                      AS tsegmentname,
    (i.init_out - ISNULL(cc.cum_out_rub, 0))                                  AS out_rub,
    CAST(
        (i.init_trf_num - ISNULL(cc.cum_trf_num, 0)) 
        / NULLIF(i.init_trf_den - ISNULL(cc.cum_trf_den, 0), 0)
        AS DECIMAL(12,6)
    ) AS rate_trf_srvz,
    CAST(
        (i.init_con_num - ISNULL(cc.cum_con_num, 0)) 
        / NULLIF(i.init_con_den - ISNULL(cc.cum_con_den, 0), 0)
        AS DECIMAL(12,6)
    ) AS rate_con_srvz
FROM #grid g
JOIN #init i
  ON i.TSegmentname = g.TSegmentname
OUTER APPLY (
    SELECT TOP (1) *
    FROM #closings_cum x
    WHERE x.TSegmentname = g.TSegmentname
      AND x.d <= g.d
    ORDER BY x.d DESC
) AS cc
ORDER BY g.d, g.TSegmentname;

/* ===== 6b) РЕЗУЛЬТАТ 2: Выходы (включая 2025-08-31) ===== */
SELECT
    g.d                                 AS [date],
    g.TSegmentname                      AS tsegmentname,
    c.out_rub_close                     AS out_rub,
    CAST(c.trf_num / NULLIF(c.trf_den, 0) AS DECIMAL(12,6)) AS rate_trf_srvz,
    CAST(c.con_num / NULLIF(c.con_den, 0) AS DECIMAL(12,6)) AS rate_con_srvz
FROM #grid g
LEFT JOIN #closings c
       ON c.TSegmentname = g.TSegmentname
      AND c.d            = g.d
ORDER BY g.d, g.TSegmentname;
