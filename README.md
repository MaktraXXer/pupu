DECLARE @dt_rep   date = '2026-01-31';  -- дата баланса
DECLARE @dt_from  date = '2026-02-01';  -- начало окна плановых выходов
DECLARE @dt_to    date = '2026-02-05';  -- конец окна плановых выходов

DROP TABLE IF EXISTS #t;

SELECT
      t.dt_rep
    , t.cli_id
    , t.con_id
    , t.con_no
    , t.SECTION_NAME
    , t.is_floatrate
    , t.TSEGMENTNAME
    , t.PROD_NAME_RES
    , t.OUT_RUB
    , t.rate_con
    , t.rate_trf
    , CAST(t.dt_close_plan AS date)  AS dt_close_plan
    , CASE
        WHEN t.rate_con >= 0.163 AND t.rate_con <= 0.17
         AND t.termdays BETWEEN 85 AND 98
        THEN 1 ELSE 0
      END AS is_promo_3m
INTO #t
FROM [ALM].[VW_balance_rest_all] t WITH (NOLOCK)
WHERE 1=1
  AND t.dt_rep = @dt_rep
  AND t.OUT_RUB IS NOT NULL
  AND t.od_flag = 1
  AND t.block_name = 'Привлечение ФЛ'
  AND t.cur = '810';


DROP TABLE IF EXISTS #cli_seg;

SELECT
      dt_rep
    , cli_id
    , CASE
        WHEN MAX(CASE WHEN TSEGMENTNAME IN (N'ДЧБО') THEN 1 ELSE 0 END) = 1 THEN N'ДЧБО'
        ELSE N'Розница'
      END AS [client_segment]
INTO #cli_seg
FROM #t
GROUP BY dt_rep, cli_id;


/*Клиенты, у которых есть плановые выходы в окне @dt_from..@dt_to, выходы только любых срочных рублевых вкладов
*/
DROP TABLE IF EXISTS #exit_clients;
SELECT DISTINCT
      t.dt_rep
    , t.cli_id
    , s.[client_segment]
INTO #exit_clients
FROM #t t
JOIN #cli_seg s
  ON s.dt_rep = t.dt_rep
 AND s.cli_id = t.cli_id
WHERE t.SECTION_NAME = N'Срочные '
  AND t.dt_close_plan BETWEEN @dt_from AND @dt_to;


/*Клиентская витрина с агрегатами нужными*/
DROP TABLE IF EXISTS #client;

SELECT
      t.dt_rep
    , t.cli_id
    , ec.[client_segment]
    -- базовые балансы на дату @dt_rep (по всем продуктам клиента которые не так интересны)
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_floatrate = 1 THEN t.OUT_RUB ELSE 0 END) AS out_float_term_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'Накопительный счёт' THEN t.OUT_RUB ELSE 0 END)             AS out_savings_total_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'До востребования'  THEN t.OUT_RUB ELSE 0 END)             AS out_demand_total_rub
    --Промо-вклады - разбивка по каждому дню окна + "прочие даты"
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 1 AND t.dt_close_plan = DATEADD(day, 0, @dt_from) THEN t.OUT_RUB ELSE 0 END) AS promo_exit_d1_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 1 AND t.dt_close_plan = DATEADD(day, 1, @dt_from) THEN t.OUT_RUB ELSE 0 END) AS promo_exit_d2_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 1 AND t.dt_close_plan = DATEADD(day, 2, @dt_from) THEN t.OUT_RUB ELSE 0 END) AS promo_exit_d3_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 1 AND t.dt_close_plan = DATEADD(day, 3, @dt_from) THEN t.OUT_RUB ELSE 0 END) AS promo_exit_d4_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 1 AND t.dt_close_plan = DATEADD(day, 4, @dt_from) THEN t.OUT_RUB ELSE 0 END) AS promo_exit_d5_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 1
               AND (t.dt_close_plan NOT BETWEEN @dt_from AND @dt_to OR t.dt_close_plan IS NULL)
               THEN t.OUT_RUB ELSE 0 END) AS promo_exit_other_rub
    --НЕ Промо-вклады - разбивка по каждому дню окна + "прочие даты"
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 0 AND t.dt_close_plan = DATEADD(day, 0, @dt_from) THEN t.OUT_RUB ELSE 0 END) AS nonpromo_exit_d1_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 0 AND t.dt_close_plan = DATEADD(day, 1, @dt_from) THEN t.OUT_RUB ELSE 0 END) AS nonpromo_exit_d2_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 0 AND t.dt_close_plan = DATEADD(day, 2, @dt_from) THEN t.OUT_RUB ELSE 0 END) AS nonpromo_exit_d3_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 0 AND t.dt_close_plan = DATEADD(day, 3, @dt_from) THEN t.OUT_RUB ELSE 0 END) AS nonpromo_exit_d4_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 0 AND t.dt_close_plan = DATEADD(day, 4, @dt_from) THEN t.OUT_RUB ELSE 0 END) AS nonpromo_exit_d5_rub
    , SUM(CASE WHEN t.SECTION_NAME = N'Срочные ' AND t.is_promo_3m = 0
               AND (t.dt_close_plan NOT BETWEEN @dt_from AND @dt_to OR t.dt_close_plan IS NULL)
               THEN t.OUT_RUB ELSE 0 END) AS nonpromo_exit_other_rub
INTO #client
FROM #t t
JOIN #exit_clients ec
  ON ec.dt_rep = t.dt_rep
 AND ec.cli_id = t.cli_id
GROUP BY t.dt_rep, t.cli_id, ec.[client_segment];


/*Сохраняем
*/
USE [ALM_TEST];
GO
DROP TABLE IF EXISTS [WORK].[temp_client_exit_0201_0205];

SELECT *
INTO [WORK].[temp_client_exit_0201_0205]
FROM #client;
