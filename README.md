/* ПАРАМЕТРЫ */
DECLARE @DateFrom date = '2025-08-01';
DECLARE @DateTo   date = CONVERT(date, GETDATE());

/* Справочник продуктов (DISTINCT) */
;WITH prod_ref AS (
    SELECT DISTINCT prod_name
    FROM ALM_TEST.markets.prod_term_rates
)
SELECT
    t.dt_rep,
    SUM(t.out_rub) AS out_rub_total,

    /* 1) СВС эффективной ставки */
    SUM(
        CASE
            WHEN t.rate_trf IS NOT NULL AND t.rate_con IS NOT NULL THEN
                (
                    CASE
                        WHEN t.rate_trf >= t.rate_con + 0.0048
                            THEN t.rate_trf                               -- категория 1
                        ELSE
                            CASE
                                WHEN pr.prod_name IS NULL                  -- продукта НЕТ в справочнике
                                    THEN t.rate_con + 0.0048 + 0.0010      -- категория 2
                                ELSE t.rate_trf                             -- продукт есть -> не применяем кат.2
                            END
                    END
                ) * t.out_rub
            ELSE 0
        END
    )
    / NULLIF(SUM(CASE WHEN t.rate_trf IS NOT NULL AND t.rate_con IS NOT NULL THEN t.out_rub ELSE 0 END), 0)
    AS eff_rate_wavg,

    /* 2) Доля объёма в категории 2 */
    SUM(
        CASE
            WHEN t.rate_trf IS NOT NULL AND t.rate_con IS NOT NULL
             AND t.rate_trf <= t.rate_con + 0.0048
             AND pr.prod_name IS NULL
                THEN t.out_rub
            ELSE 0
        END
    )
    / NULLIF(SUM(CASE WHEN t.rate_trf IS NOT NULL AND t.rate_con IS NOT NULL THEN t.out_rub ELSE 0 END), 0)
    AS share_cat2_by_volume,

    /* 3) «Вычисленный трансферт» без правил */
    SUM(CASE WHEN t.rate_trf IS NOT NULL THEN t.rate_trf * t.out_rub ELSE 0 END)
    / NULLIF(SUM(CASE WHEN t.rate_trf IS NOT NULL THEN t.out_rub ELSE 0 END), 0)
    AS rate_trf_wavg_plain

FROM alm.[ALM].[vw_balance_rest_all] AS t WITH (NOLOCK)
LEFT JOIN prod_ref pr
       ON pr.prod_name = t.prod_name_res
WHERE
    t.dt_rep BETWEEN @DateFrom AND @DateTo
    AND t.dt_open  = t.dt_rep
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag      = 1
    AND t.is_floatrate = 0
    AND t.cur          = '810'
    AND t.out_rub IS NOT NULL
GROUP BY t.dt_rep
OPTION (RECOMPILE);
