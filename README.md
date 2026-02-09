/* =========================================================
   0) БАЗА: клиентская витрина на 31.01 уже сохранена сюда:
      [ALM_TEST].[WORK].[temp_client_exit_0201_0205]
   ========================================================= */

USE [ALM_TEST];
GO

/* =========================================================
   1) 31 ЯНВАРЯ: суммарные балансы (в млрд) по клиентам когорты
      + сколько по плану выходит PROMO / NON-PROMO (в млрд)
      за окно 01–05.02 и за "прочие даты"
   ========================================================= */
SELECT
      MIN(dt_rep) AS dt_rep
    , COUNT(*)    AS clients_cnt

    , SUM(out_savings_total_rub) / 1e9 AS ns_31jan_rub_bln
    , SUM(out_demand_total_rub)  / 1e9 AS dvs_31jan_rub_bln
    , SUM(out_float_term_rub)    / 1e9 AS float_term_31jan_rub_bln

    , SUM(promo_exit_d1_rub + promo_exit_d2_rub + promo_exit_d3_rub + promo_exit_d4_rub + promo_exit_d5_rub) / 1e9 AS promo_exit_0105_rub_bln
    , SUM(promo_exit_other_rub)  / 1e9 AS promo_exit_other_rub_bln

    , SUM(nonpromo_exit_d1_rub + nonpromo_exit_d2_rub + nonpromo_exit_d3_rub + nonpromo_exit_d4_rub + nonpromo_exit_d5_rub) / 1e9 AS nonpromo_exit_0105_rub_bln
    , SUM(nonpromo_exit_other_rub) / 1e9 AS nonpromo_exit_other_rub_bln
FROM [WORK].[temp_client_exit_0201_0205];


/* =========================================================
   2) ДЕТАЛИЗАЦИЯ ПО ДНЯМ: сколько должно выходить в каждый день
      PROMO / NON-PROMO (в млрд)
   ========================================================= */
SELECT
      '2026-02-01' AS exit_day
    , SUM(promo_exit_d1_rub)    / 1e9 AS promo_rub_bln
    , SUM(nonpromo_exit_d1_rub) / 1e9 AS nonpromo_rub_bln
    , (SUM(promo_exit_d1_rub) + SUM(nonpromo_exit_d1_rub)) / 1e9 AS total_rub_bln
FROM [WORK].[temp_client_exit_0201_0205]
UNION ALL
SELECT
      '2026-02-02'
    , SUM(promo_exit_d2_rub)    / 1e9
    , SUM(nonpromo_exit_d2_rub) / 1e9
    , (SUM(promo_exit_d2_rub) + SUM(nonpromo_exit_d2_rub)) / 1e9
FROM [WORK].[temp_client_exit_0201_0205]
UNION ALL
SELECT
      '2026-02-03'
    , SUM(promo_exit_d3_rub)    / 1e9
    , SUM(nonpromo_exit_d3_rub) / 1e9
    , (SUM(promo_exit_d3_rub) + SUM(nonpromo_exit_d3_rub)) / 1e9
FROM [WORK].[temp_client_exit_0201_0205]
UNION ALL
SELECT
      '2026-02-04'
    , SUM(promo_exit_d4_rub)    / 1e9
    , SUM(nonpromo_exit_d4_rub) / 1e9
    , (SUM(promo_exit_d4_rub) + SUM(nonpromo_exit_d4_rub)) / 1e9
FROM [WORK].[temp_client_exit_0201_0205]
UNION ALL
SELECT
      '2026-02-05'
    , SUM(promo_exit_d5_rub)    / 1e9
    , SUM(nonpromo_exit_d5_rub) / 1e9
    , (SUM(promo_exit_d5_rub) + SUM(nonpromo_exit_d5_rub)) / 1e9
FROM [WORK].[temp_client_exit_0201_0205]
ORDER BY exit_day;


/* =========================================================
   3) 06 ФЕВРАЛЯ: суммарные балансы этих же клиентов на 06.02
      (НС, ДВС, плавающие вклады, обычные вклады)
      ВАЖНО: "обычные вклады" = Срочные НЕ плавающие
      (в млрд)
   ========================================================= */

DROP TABLE IF EXISTS #b0602;

SELECT
      t.dt_rep
    , t.cli_id
    , t.SECTION_NAME
    , t.is_floatrate
    , t.OUT_RUB
INTO #b0602
FROM [ALM].[VW_balance_rest_all] t WITH (NOLOCK)
JOIN [WORK].[temp_client_exit_0201_0205] c
  ON c.cli_id = t.cli_id
WHERE 1=1
  AND t.dt_rep = '20260206'
  AND t.OUT_RUB IS NOT NULL
  AND t.od_flag = 1
  AND t.block_name = 'Привлечение ФЛ'
  AND t.cur = '810';


SELECT
      '2026-02-06' AS dt_rep
    , COUNT(DISTINCT cli_id) AS clients_cnt

    , SUM(CASE WHEN SECTION_NAME = N'Накопительный счёт' THEN OUT_RUB ELSE 0 END) / 1e9 AS ns_06feb_rub_bln
    , SUM(CASE WHEN SECTION_NAME = N'До востребования'  THEN OUT_RUB ELSE 0 END) / 1e9 AS dvs_06feb_rub_bln

    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_floatrate = 1 THEN OUT_RUB ELSE 0 END) / 1e9 AS float_term_06feb_rub_bln
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND ISNULL(is_floatrate,0) = 0 THEN OUT_RUB ELSE 0 END) / 1e9 AS term_fixed_06feb_rub_bln

    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' THEN OUT_RUB ELSE 0 END) / 1e9 AS term_total_06feb_rub_bln
    , SUM(OUT_RUB) / 1e9 AS total_06feb_rub_bln
FROM #b0602;
