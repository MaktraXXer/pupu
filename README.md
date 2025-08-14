WITH j AS (
  SELECT
    vol_31_close_aug     = SUM(CASE WHEN dt_close BETWEEN @CloseFrom AND @CloseTo THEN vol31 ELSE 0 END),
    vol_31_not_close_aug = SUM(CASE WHEN dt_close IS NULL OR dt_close < @CloseFrom OR dt_close > @CloseTo THEN vol31 ELSE 0 END)
  FROM #july31
),
a AS (
  SELECT
    vol_12_opened_aug_to_chk = SUM(CASE WHEN a12.dt_open BETWEEN @CloseFrom AND @ChkDate THEN a12.vol12 ELSE 0 END),
    vol_12_from_31_alive     = SUM(CASE WHEN j31.con_id IS NOT NULL THEN a12.vol12 ELSE 0 END)
  FROM #aug12 a12
  LEFT JOIN #july31 j31 ON j31.con_id = a12.con_id  -- «остался жить» = тот же договор был 31.07
)
SELECT
  -- 31.07
  j.vol_31_close_aug,
  j.vol_31_not_close_aug,
  total_31 = j.vol_31_close_aug + j.vol_31_not_close_aug,
  -- 12.08
  a.vol_12_opened_aug_to_chk,
  a.vol_12_from_31_alive,
  total_1208 = a.vol_12_opened_aug_to_chk + a.vol_12_from_31_alive
FROM j CROSS JOIN a
OPTION (RECOMPILE);
