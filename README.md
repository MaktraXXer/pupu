DECLARE @dt_rep date = '2025-08-31';

-- 1) Снимок портфеля на дату отчёта
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
SELECT
    t.TSegmentname,
    CAST(t.dt_close AS date) AS dt_close_d,
    t.out_rub,
    t.rate_con,
    t.rate_trf
INTO #base
FROM alm.[ALM].[vw_balance_rest_all] t WITH (NOLOCK)
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

-- 2) Диапазон дат
DECLARE @d_end date;
SELECT @d_end = ISNULL(MAX(dt_close_d), @dt_rep) FROM #base;

-- 3) Календарь рекурсией (@dt_rep..@d_end)
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
;WITH cal AS (
    SELECT @dt_rep AS d
    UNION ALL
    SELECT DATEADD(DAY, 1, d)
    FROM cal
    WHERE d < @d_end
)
SELECT d INTO #cal FROM cal
OPTION (MAXRECURSION 0);

-- 4) Сегменты × календарь (чтобы были строки на каждый день)
IF OBJECT_ID('tempdb..#grid') IS NOT NULL DROP TABLE #grid;
SELECT s.TSegmentname, c.d
INTO #grid
FROM (SELECT DISTINCT TSegmentname FROM #base) s
CROSS JOIN #cal c;

-- 5) Выходы по датам (дельты закрытий)
IF OBJECT_ID('tempdb..#closings') IS NOT NULL DROP TABLE #closings;
SELECT
    b.TSegmentname,
    b.dt_close_d AS d,
    SUM(b.out_rub)                                                    AS out_rub_close,
    SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END) AS trf_num,
    SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub END)              AS trf_den,
    SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END) AS con_num,
    SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END)              AS con_den
INTO #closings
FROM #base b
GROUP BY b.TSegmentname, b.dt_close_d;

-- 6) Начальные суммы/числители/знаменатели на @dt_rep
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

-- 7) Амортизация: init − кумулятив закрытий (<= d)
SELECT
    g.d                                  AS [date],
    g.TSegmentname                        AS tsegmentname,
    (i.init_out
     - SUM(COALESCE(c.out_rub_close, 0)) OVER (PARTITION BY g.TSegmentname ORDER BY g.d
                                               ROWS UNBOUNDED PRECEDING)
    ) AS out_rub,
    CAST( (i.init_trf_num
           - SUM(COALESCE(c.trf_num, 0)) OVER (PARTITION BY g.TSegmentname ORDER BY g.d
                                               ROWS UNBOUNDED PRECEDING)
          )
          / NULLIF(i.init_trf_den
           - SUM(COALESCE(c.trf_den, 0)) OVER (PARTITION BY g.TSegmentname ORDER BY g.d
                                               ROWS UNBOUNDED PRECEDING), 0)
          AS DECIMAL(12,6)
    ) AS rate_trf_srvz,
    CAST( (i.init_con_num
           - SUM(COALESCE(c.con_num, 0)) OVER (PARTITION BY g.TSegmentname ORDER BY g.d
                                               ROWS UNBOUNDED PRECEDING)
          )
          / NULLIF(i.init_con_den
           - SUM(COALESCE(c.con_den, 0)) OVER (PARTITION BY g.TSegmentname ORDER BY g.d
                                               ROWS UNBOUNDED PRECEDING), 0)
          AS DECIMAL(12,6)
    ) AS rate_con_srvz
FROM #grid g
LEFT JOIN #closings c
       ON c.TSegmentname = g.TSegmentname
      AND c.d            = g.d
JOIN #init i
  ON i.TSegmentname = g.TSegmentname
ORDER BY g.d, g.TSegmentname;
