USE [ALM_TEST];
GO

/* 31 ЯНВАРЯ: в одном запросе и “итоги”, и детализация по дням (в млрд) */
SELECT
      0 AS sort_
    , N'ИТОГО 31.01 (балансы + плановые выходы)' AS row_type
    , CAST(NULL AS date) AS exit_day
    , d.dt_rep AS dt_rep
    , COUNT(*) AS clients_cnt

    , SUM(out_savings_total_rub) / 1e9 AS ns_31jan_rub_bln
    , SUM(out_demand_total_rub)  / 1e9 AS dvs_31jan_rub_bln
    , SUM(out_float_term_rub)    / 1e9 AS float_term_31jan_rub_bln

    , SUM(promo_exit_d1_rub + promo_exit_d2_rub + promo_exit_d3_rub + promo_exit_d4_rub + promo_exit_d5_rub) / 1e9 AS promo_exit_0105_rub_bln
    , SUM(promo_exit_other_rub) / 1e9 AS promo_exit_other_rub_bln

    , SUM(nonpromo_exit_d1_rub + nonpromo_exit_d2_rub + nonpromo_exit_d3_rub + nonpromo_exit_d4_rub + nonpromo_exit_d5_rub) / 1e9 AS nonpromo_exit_0105_rub_bln
    , SUM(nonpromo_exit_other_rub) / 1e9 AS nonpromo_exit_other_rub_bln

    , CAST(NULL AS float) AS promo_day_rub_bln
    , CAST(NULL AS float) AS nonpromo_day_rub_bln
    , CAST(NULL AS float) AS total_day_rub_bln
FROM [WORK].[temp_client_exit_0201_0205] c
CROSS JOIN (SELECT DISTINCT dt_rep FROM [WORK].[temp_client_exit_0201_0205]) d
GROUP BY d.dt_rep

UNION ALL

SELECT
      1 AS sort_
    , N'ДЕТАЛИЗАЦИЯ ПО ДНЯМ (плановые выходы)' AS row_type
    , v.exit_day
    , d.dt_rep AS dt_rep
    , COUNT(*) AS clients_cnt

    , CAST(NULL AS float) AS ns_31jan_rub_bln
    , CAST(NULL AS float) AS dvs_31jan_rub_bln
    , CAST(NULL AS float) AS float_term_31jan_rub_bln

    , CAST(NULL AS float) AS promo_exit_0105_rub_bln
    , CAST(NULL AS float) AS promo_exit_other_rub_bln
    , CAST(NULL AS float) AS nonpromo_exit_0105_rub_bln
    , CAST(NULL AS float) AS nonpromo_exit_other_rub_bln

    , SUM(v.promo_rub)    / 1e9 AS promo_day_rub_bln
    , SUM(v.nonpromo_rub) / 1e9 AS nonpromo_day_rub_bln
    , (SUM(v.promo_rub) + SUM(v.nonpromo_rub)) / 1e9 AS total_day_rub_bln
FROM [WORK].[temp_client_exit_0201_0205] c
CROSS JOIN (SELECT DISTINCT dt_rep FROM [WORK].[temp_client_exit_0201_0205]) d
CROSS APPLY (VALUES
      ('2026-02-01', c.promo_exit_d1_rub, c.nonpromo_exit_d1_rub)
    , ('2026-02-02', c.promo_exit_d2_rub, c.nonpromo_exit_d2_rub)
    , ('2026-02-03', c.promo_exit_d3_rub, c.nonpromo_exit_d3_rub)
    , ('2026-02-04', c.promo_exit_d4_rub, c.nonpromo_exit_d4_rub)
    , ('2026-02-05', c.promo_exit_d5_rub, c.nonpromo_exit_d5_rub)
) v(exit_day, promo_rub, nonpromo_rub)
GROUP BY d.dt_rep, v.exit_day
ORDER BY sort_, exit_day;
