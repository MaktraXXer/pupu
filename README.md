USE [ALM_TEST];
GO

SELECT
      '2026-02-06' AS dt_rep
    , COUNT(DISTINCT t.cli_id) AS clients_cnt

    , SUM(CASE WHEN t.SECTION_NAME = N'До востребования'  THEN t.OUT_RUB ELSE 0 END) / 1e9 AS dvs_rub_bln
    , SUM(CASE WHEN t.SECTION_NAME = N'Накопительный счёт' THEN t.OUT_RUB ELSE 0 END) / 1e9 AS ns_rub_bln

    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_floatrate = 1 THEN t.OUT_RUB ELSE 0 END) / 1e9 AS float_term_rub_bln
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND ISNULL(t.is_floatrate,0) = 0 THEN t.OUT_RUB ELSE 0 END) / 1e9 AS other_term_rub_bln

    , SUM(t.OUT_RUB) / 1e9 AS total_rub_bln
FROM [ALM].[VW_balance_rest_all] t WITH (NOLOCK)
JOIN [WORK].[temp_client_exit_0201_0205] c
  ON c.cli_id = t.cli_id
WHERE 1=1
  AND t.dt_rep = '20260206'
  AND t.OUT_RUB IS NOT NULL
  AND t.od_flag = 1
  AND t.block_name = 'Привлечение ФЛ'
  AND t.cur = '810';
