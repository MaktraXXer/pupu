DECLARE @dt_rep date = '2025-08-31';

IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
IF OBJECT_ID('tempdb..#cal')  IS NOT NULL DROP TABLE #cal;
IF OBJECT_ID('tempdb..#seg')  IS NOT NULL DROP TABLE #seg;

-- 1) База: портфель на @dt_rep (cur=810) + con_id + MonthlyCONV_ForecastKeyRate из снапа @dt_rep
SELECT
    t.con_id,
    t.cur,
    CAST(t.dt_close AS date) AS dt_close_d,
    t.out_rub,
    t.rate_con,
    t.rate_trf,
    s.MonthlyCONV_ForecastKeyRate
INTO #base
FROM alm.[ALM].[vw_balance_rest_all] t WITH (NOLOCK)
LEFT JOIN (
    SELECT
        con_id,
        MAX(MonthlyCONV_ForecastKeyRate) AS MonthlyCONV_ForecastKeyRate
    FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap] WITH (NOLOCK)
    WHERE dt_rep = @dt_rep
    GROUP BY con_id
) s
    ON s.con_id = t.con_id
WHERE t.dt_rep       = @dt_rep
  AND t.AP           = N'Пассив'
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.cur          = '810'
  AND t.acc_role     = N'LIAB'
  AND t.out_rub      IS NOT NULL
  AND t.tprod_name   = N'Вклады ФЛ'
  AND t.dt_close     > @dt_rep;              -- завязали на @dt_rep

-- 2) dt_end: до какой даты считать амортизацию (включительно)
--    По умолчанию: max(dt_close) в портфеле на @dt_rep, но не раньше @dt_rep
DECLARE @dt_end date;
SELECT @dt_end = ISNULL(MAX(dt_close_d), @dt_rep) FROM #base;

-- Если хочешь руками ограничивать горизонт (пример):
-- DECLARE @dt_end date = '2025-12-31';
-- (тогда убери SELECT выше)

-- 3) Календарь от @dt_rep до @dt_end (включительно)
;WITH cal AS (
    SELECT @dt_rep AS d
    UNION ALL
    SELECT DATEADD(DAY, 1, d) FROM cal WHERE d < @dt_end
)
SELECT d INTO #cal FROM cal
OPTION (MAXRECURSION 0);

-- 4) "Сегменты" (тут один cur, оставим общий вид)
SELECT DISTINCT cur INTO #seg FROM #base;

-- 5) Амортизация: объём, который "выбывает" В КАЖДУЮ ДАТУ (dt_close_d = date),
--    считаем только в диапазоне [@dt_rep; @dt_end]
SELECT
    c.d AS [date],
    s.cur AS cur,

    COALESCE(SUM(CASE WHEN b.dt_close_d = c.d THEN b.out_rub END), 0) AS amort_out_rub,

    CAST(
        SUM(CASE WHEN b.dt_close_d = c.d AND b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END)
        / NULLIF(SUM(CASE WHEN b.dt_close_d = c.d AND b.rate_trf IS NOT NULL THEN b.out_rub END), 0)
        AS DECIMAL(12,6)
    ) AS amort_rate_trf_srvz,

    CAST(
        SUM(CASE WHEN b.dt_close_d = c.d AND b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END)
        / NULLIF(SUM(CASE WHEN b.dt_close_d = c.d AND b.rate_con IS NOT NULL THEN b.out_rub END), 0)
        AS DECIMAL(12,6)
    ) AS amort_rate_con_srvz,

    CAST(
        SUM(CASE WHEN b.dt_close_d = c.d AND b.MonthlyCONV_ForecastKeyRate IS NOT NULL
                 THEN b.out_rub * b.MonthlyCONV_ForecastKeyRate END)
        / NULLIF(SUM(CASE WHEN b.dt_close_d = c.d AND b.MonthlyCONV_ForecastKeyRate IS NOT NULL
                          THEN b.out_rub END), 0)
        AS DECIMAL(12,6)
    ) AS amort_MonthlyCONV_ForecastKeyRate_srvz

FROM #cal c
CROSS JOIN #seg s
LEFT JOIN #base b
    ON b.cur = s.cur
GROUP BY c.d, s.cur
ORDER BY c.d, s.cur;

Если тебе нужно, чтобы @dt_end задавался явно параметром (а не как max dt_close), скажи какую дату/горизонт хочешь по умолчанию (например, EOMONTH(@dt_rep, 12) или фикс “+365 дней”).
