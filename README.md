DECLARE @dt_rep date = '2025-08-31';

-- 1) Снимок портфеля
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

-- 3) Календарь (простая рекурсия или готовая таблица дат)
;WITH cal AS (
    SELECT @dt_rep AS d
    UNION ALL
    SELECT DATEADD(DAY, 1, d) FROM cal WHERE d < @d_end
)
SELECT d INTO #cal FROM cal
OPTION (MAXRECURSION 0);

-- 4a) Амортизация: на каждую дату берём ещё живые вклады
SELECT
    c.d AS [date],
    b.TSegmentname,
    SUM(b.out_rub)                            AS out_rub,
    CAST(SUM(b.out_rub * b.rate_trf) / NULLIF(SUM(b.out_rub),0) AS DECIMAL(12,6)) AS rate_trf_srvz,
    CAST(SUM(b.out_rub * b.rate_con) / NULLIF(SUM(b.out_rub),0) AS DECIMAL(12,6)) AS rate_con_srvz
FROM #cal c
JOIN #base b
  ON b.dt_close_d > c.d
GROUP BY c.d, b.TSegmentname
ORDER BY c.d, b.TSegmentname;

-- 4b) Выходы: на каждую дату берём вклады с закрытием ровно в этот день
SELECT
    c.d AS [date],
    b.TSegmentname,
    SUM(b.out_rub)                            AS out_rub,
    CAST(SUM(b.out_rub * b.rate_trf) / NULLIF(SUM(b.out_rub),0) AS DECIMAL(12,6)) AS rate_trf_srvz,
    CAST(SUM(b.out_rub * b.rate_con) / NULLIF(SUM(b.out_rub),0) AS DECIMAL(12,6)) AS rate_con_srvz
FROM #cal c
LEFT JOIN #base b
  ON b.dt_close_d = c.d
GROUP BY c.d, b.TSegmentname
ORDER BY c.d, b.TSegmentname;
