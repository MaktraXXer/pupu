/* ПАРАМЕТРЫ */
DECLARE @DateFrom date = '2025-08-01';
DECLARE @DateTo   date = CONVERT(date, GETDATE());

/* НОВЫЕ СДЕЛКИ: только плавающая ставка (is_floatrate=1), без доп. правил */
SELECT
    t.dt_rep,
    SUM(t.out_rub) AS out_rub_total,

    /* СВС по transfer-ставке (только где rate_trf задан) */
    SUM(CASE WHEN t.rate_trf IS NOT NULL THEN t.rate_trf * t.out_rub ELSE 0 END)
      / NULLIF(SUM(CASE WHEN t.rate_trf IS NOT NULL THEN t.out_rub ELSE 0 END), 0)
      AS rate_trf_wavg

FROM alm.[ALM].[vw_balance_rest_all] AS t WITH (NOLOCK)
WHERE
    t.dt_rep BETWEEN @DateFrom AND @DateTo
    AND t.dt_open = CAST(t.dt_rep AS datetime)   -- ровно полночь того же дня
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag      = 1
    AND t.is_floatrate = 1                       -- ← плавающая ставка
    AND t.cur          = '810'
    AND t.out_rub IS NOT NULL
GROUP BY t.dt_rep
OPTION (RECOMPILE);
