/* ПАРАМЕТРЫ ---------------------------------------------------------*/
DECLARE @DateFrom date = '2025-08-01';
DECLARE @DateTo   date = CONVERT(date, GETDATE());

/* СПРАВОЧНЫЙ КАЛЕНДАРЬ --------------------------------------------*/
WITH days AS (
    SELECT @DateFrom AS dt_rep
    UNION ALL
    SELECT DATEADD(day, 1, dt_rep)
    FROM   days
    WHERE  dt_rep < @DateTo
),
/* БАЗА: только «новые» вклады в день dt_rep + фильтры --------------*/
base AS (
    SELECT
        t.dt_rep,
        t.out_rub,
        t.rate_trf,
        t.rate_con
    FROM alm.[ALM].[vw_balance_rest_all] t WITH (NOLOCK)
    WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
      AND t.dt_open  = t.dt_rep                 -- новые в день dt_rep
      AND t.section_name = N'Срочные'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.is_floatrate = 0
      AND t.cur          = '810'
      AND t.out_rub IS NOT NULL
),
/* ДОБАВИМ эффективную ставку и метки категорий ---------------------*/
prepared AS (
    SELECT
        b.dt_rep,
        b.out_rub,
        b.rate_trf,
        b.rate_con,
        CASE
            WHEN b.rate_trf IS NOT NULL AND b.rate_con IS NOT NULL THEN
                CASE
                    WHEN b.rate_trf >= b.rate_con + 0.0048 THEN b.rate_trf              -- категория 1
                    ELSE                                 b.rate_con + 0.0048 + 0.0010  -- категория 2 (<=)
                END
            ELSE NULL
        END AS eff_rate,
        CASE
            WHEN b.rate_trf IS NOT NULL AND b.rate_con IS NOT NULL
                 AND b.rate_trf <= b.rate_con + 0.0048
            THEN 1 ELSE 0
        END AS is_cat2,
        CASE
            WHEN b.rate_trf IS NOT NULL AND b.rate_con IS NOT NULL THEN 1 ELSE 0
        END AS is_valid_for_wavg
    FROM base b
),
/* АГРЕГИРОВАНИЕ по дню ---------------------------------------------*/
agg AS (
    SELECT
        p.dt_rep,
        SUM(p.out_rub) AS out_rub_total,                                    -- общий объём «новых» за день
        SUM(CASE WHEN p.is_valid_for_wavg = 1 THEN p.out_rub ELSE 0 END) AS vol_valid,     -- объём, попавший в СВС
        SUM(CASE WHEN p.is_valid_for_wavg = 1 THEN p.eff_rate * p.out_rub ELSE 0 END)
            / NULLIF(SUM(CASE WHEN p.is_valid_for_wavg = 1 THEN p.out_rub ELSE 0 END), 0)
            AS eff_rate_wavg,                                               -- СВС эффективной ставки
        SUM(CASE WHEN p.is_valid_for_wavg = 1 AND p.is_cat2 = 1 THEN p.out_rub ELSE 0 END)
            / NULLIF(SUM(CASE WHEN p.is_valid_for_wavg = 1 THEN p.out_rub ELSE 0 END), 0)
            AS share_cat2_by_volume                                         -- доля объёма в категории 2
    FROM prepared p
    GROUP BY p.dt_rep
)
/* ФИНАЛ ------------------------------------------------------------*/
SELECT
    d.dt_rep,
    COALESCE(a.out_rub_total, 0.0)  AS out_rub_total,
    a.eff_rate_wavg                 AS eff_rate_wavg,
    a.share_cat2_by_volume          AS share_cat2_by_volume
    -- , a.vol_valid                 AS out_rub_used_for_wavg  -- опционально: объём, попавший в СВС
FROM days d
LEFT JOIN agg a
  ON a.dt_rep = d.dt_rep
OPTION (MAXRECURSION 0, RECOMPILE);
