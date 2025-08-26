/* ПАРАМЕТРЫ */
DECLARE @DateFrom date = '2025-08-01';
DECLARE @DateTo   date = CONVERT(date, GETDATE());

/* ОДИН ПРОХОД С АГРЕГАЦИЕЙ */
SELECT
    t.dt_rep,
    SUM(t.out_rub) AS out_rub_total,

    /* 1) Эффективная ставка по твоим двум правилам */
    SUM(
        CASE
            WHEN t.rate_trf IS NOT NULL AND t.rate_con IS NOT NULL THEN
                (CASE
                    WHEN t.rate_trf >= t.rate_con + 0.0048
                        THEN t.rate_trf                 -- категория 1
                    ELSE t.rate_con + 0.0048 + 0.0010  -- категория 2
                 END) * t.out_rub
            ELSE 0
        END
    )
    / NULLIF(SUM(CASE WHEN t.rate_trf IS NOT NULL AND t.rate_con IS NOT NULL THEN t.out_rub ELSE 0 END), 0)
    AS eff_rate_wavg,

    /* 2) Доля объёма в категории 2 (в знаменателе — тот же валидный объём) */
    SUM(
        CASE
            WHEN t.rate_trf IS NOT NULL AND t.rate_con IS NOT NULL
                 AND t.rate_trf <= t.rate_con + 0.0048
                THEN t.out_rub
            ELSE 0
        END
    )
    / NULLIF(SUM(CASE WHEN t.rate_trf IS NOT NULL AND t.rate_con IS NOT NULL THEN t.out_rub ELSE 0 END), 0)
    AS share_cat2_by_volume,

    /* 3) «Вычисленный трансферт» без учёта правил (просто СВС rate_trf по тем, где он задан) */
    SUM(CASE WHEN t.rate_trf IS NOT NULL THEN t.rate_trf * t.out_rub ELSE 0 END)
    / NULLIF(SUM(CASE WHEN t.rate_trf IS NOT NULL THEN t.out_rub ELSE 0 END), 0)
    AS rate_trf_wavg_plain

FROM alm.[ALM].[vw_balance_rest_all] AS t WITH (NOLOCK)
WHERE
    t.dt_rep BETWEEN @DateFrom AND @DateTo
    AND t.dt_open  = t.dt_rep              -- только новые за день
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag      = 1
    AND t.is_floatrate = 0
    AND t.cur          = '810'
    AND t.out_rub IS NOT NULL
GROUP BY t.dt_rep
OPTION (RECOMPILE);
