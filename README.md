-- 1. Существующие клиенты (как у тебя)
SELECT
    'core' AS client_type,
    COALESCE(a.prod_id, b.prod_id) AS prod_id,

    -- НС
    a.ns_out_rub_0730,
    b.ns_out_rub_0208,
    b.ns_out_rub_0208 - a.ns_out_rub_0730 AS delta_ns_out_rub,

    CASE WHEN a.ns_base_0730 > 0 THEN a.ns_weighted_rate_0730 / a.ns_base_0730 ELSE NULL END AS ns_avg_rate_0730,
    CASE WHEN b.ns_base_0208 > 0 THEN b.ns_weighted_rate_0208 / b.ns_base_0208 ELSE NULL END AS ns_avg_rate_0208,
    CASE 
        WHEN a.ns_base_0730 > 0 AND b.ns_base_0208 > 0 THEN 
            (b.ns_weighted_rate_0208 / b.ns_base_0208) - (a.ns_weighted_rate_0730 / a.ns_base_0730)
        ELSE NULL
    END AS delta_ns_rate,

    -- Вклады
    dep_0730.dep_out_rub_0730,
    dep_0208.dep_out_rub_0208,
    dep_0208.dep_out_rub_0208 - dep_0730.dep_out_rub_0730 AS delta_dep_out_rub,

    CASE WHEN dep_0730.dep_base_0730 > 0 THEN dep_0730.dep_weighted_rate_0730 / dep_0730.dep_base_0730 ELSE NULL END AS dep_avg_rate_0730,
    CASE WHEN dep_0208.dep_base_0208 > 0 THEN dep_0208.dep_weighted_rate_0208 / dep_0208.dep_base_0208 ELSE NULL END AS dep_avg_rate_0208,
    CASE 
        WHEN dep_0730.dep_base_0730 > 0 AND dep_0208.dep_base_0208 > 0 THEN
            (dep_0208.dep_weighted_rate_0208 / dep_0208.dep_base_0208) - (dep_0730.dep_weighted_rate_0730 / dep_0730.dep_base_0730)
        ELSE NULL
    END AS delta_dep_rate,

    CASE WHEN dep_0730.dep_base_0730 > 0 THEN dep_0730.dep_weighted_term_0730 / dep_0730.dep_base_0730 ELSE NULL END AS dep_avg_term_0730,
    CASE WHEN dep_0208.dep_base_0208 > 0 THEN dep_0208.dep_weighted_term_0208 / dep_0208.dep_base_0208 ELSE NULL END AS dep_avg_term_0208,
    CASE 
        WHEN dep_0730.dep_base_0730 > 0 AND dep_0208.dep_base_0208 > 0 THEN
            (dep_0208.dep_weighted_term_0208 / dep_0208.dep_base_0208) - (dep_0730.dep_weighted_term_0730 / dep_0730.dep_base_0730)
        ELSE NULL
    END AS delta_dep_term

FROM agg_ns_0730 a
FULL OUTER JOIN agg_ns_0208 b ON a.prod_id = b.prod_id
CROSS JOIN agg_dep_0730 dep_0730
CROSS JOIN agg_dep_0208 dep_0208

UNION ALL

-- 2. Новые клиенты (были на 02.08, не было на 30.07)
SELECT
    'new' AS client_type,
    prod_id,

    -- НС: на 30.07 — NULL
    NULL AS ns_out_rub_0730,
    SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) AS ns_out_rub_0208,
    NULL AS delta_ns_out_rub,

    NULL AS ns_avg_rate_0730,
    CASE 
        WHEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) > 0
        THEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB * rate_con ELSE 0 END)
             / NULLIF(SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END), 0)
        ELSE NULL
    END AS ns_avg_rate_0208,
    NULL AS delta_ns_rate,

    -- Вклады: на 30.07 — NULL
    NULL AS dep_out_rub_0730,
    SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) AS dep_out_rub_0208,
    NULL AS delta_dep_out_rub,

    NULL AS dep_avg_rate_0730,
    CASE 
        WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
        THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * rate_con ELSE 0 END)
             / NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
        ELSE NULL
    END AS dep_avg_rate_0208,
    NULL AS delta_dep_rate,

    NULL AS dep_avg_term_0730,
    CASE 
        WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
        THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * termdays ELSE 0 END)
             / NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
        ELSE NULL
    END AS dep_avg_term_0208,
    NULL AS delta_dep_term

FROM ALM.vw_balance_rest_all d WITH (NOLOCK)
WHERE 
    d.dt_rep = '2025-08-02'
    AND d.od_flag = 1
    AND d.section_name IN ('Накопительный счёт', 'Срочные')
    AND d.block_name = N'Привлечение ФЛ'
    AND NOT EXISTS (
        SELECT 1
        FROM ALM.vw_balance_rest_all prev WITH (NOLOCK)
        WHERE 
            prev.cli_id = d.cli_id
            AND prev.dt_rep = '2025-07-30'
            AND prev.od_flag = 1
    )
GROUP BY prod_id
