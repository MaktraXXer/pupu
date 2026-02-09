/* 4) Сохранение #client в ALM_TEST.WORK */
USE [ALM_TEST];
GO

DROP TABLE IF EXISTS [WORK].[temp_client_exit_0201_0205];

SELECT *
INTO [WORK].[temp_client_exit_0201_0205]
FROM #client;


/* 5) Суммарная статистика по витрине */
SELECT
      MIN(dt_rep)                                   AS dt_rep
    , COUNT(*)                                      AS clients_cnt

    , SUM(out_savings_total_rub)   / 1e6            AS ns_total_rub_mln
    , SUM(out_demand_total_rub)    / 1e6            AS dvs_total_rub_mln
    , SUM(out_float_term_rub)      / 1e6            AS float_term_total_rub_mln

    , SUM(promo_exit_d1_rub)       / 1e6            AS promo_exit_d1_rub_mln
    , SUM(promo_exit_d2_rub)       / 1e6            AS promo_exit_d2_rub_mln
    , SUM(promo_exit_d3_rub)       / 1e6            AS promo_exit_d3_rub_mln
    , SUM(promo_exit_d4_rub)       / 1e6            AS promo_exit_d4_rub_mln
    , SUM(promo_exit_d5_rub)       / 1e6            AS promo_exit_d5_rub_mln
    , SUM(promo_exit_other_rub)    / 1e6            AS promo_exit_other_rub_mln
    , SUM(promo_exit_d1_rub + promo_exit_d2_rub + promo_exit_d3_rub + promo_exit_d4_rub + promo_exit_d5_rub) / 1e6
                                                     AS promo_exit_window_rub_mln

    , SUM(nonpromo_exit_d1_rub)    / 1e6            AS nonpromo_exit_d1_rub_mln
    , SUM(nonpromo_exit_d2_rub)    / 1e6            AS nonpromo_exit_d2_rub_mln
    , SUM(nonpromo_exit_d3_rub)    / 1e6            AS nonpromo_exit_d3_rub_mln
    , SUM(nonpromo_exit_d4_rub)    / 1e6            AS nonpromo_exit_d4_rub_mln
    , SUM(nonpromo_exit_d5_rub)    / 1e6            AS nonpromo_exit_d5_rub_mln
    , SUM(nonpromo_exit_other_rub) / 1e6            AS nonpromo_exit_other_rub_mln
    , SUM(nonpromo_exit_d1_rub + nonpromo_exit_d2_rub + nonpromo_exit_d3_rub + nonpromo_exit_d4_rub + nonpromo_exit_d5_rub) / 1e6
                                                     AS nonpromo_exit_window_rub_mln

    , SUM(
        promo_exit_d1_rub + promo_exit_d2_rub + promo_exit_d3_rub + promo_exit_d4_rub + promo_exit_d5_rub
      + nonpromo_exit_d1_rub + nonpromo_exit_d2_rub + nonpromo_exit_d3_rub + nonpromo_exit_d4_rub + nonpromo_exit_d5_rub
      ) / 1e6                                       AS total_exit_window_rub_mln
FROM [WORK].[temp_client_exit_0201_0205];
