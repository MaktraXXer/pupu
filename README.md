/* 1) Посделочное полотно на dt_rep = 2026-01-31 */
DROP TABLE IF EXISTS #t;

SELECT
      t.dt_rep
    , t.cli_id
    , t.con_id
    , t.con_no
    , t.SECTION_NAME                 -- N'Срочные ', N'До востребования', N'Накопительный счёт'
    , t.is_floatrate                 -- флаг плавающих (может быть и на НС)
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
  AND t.dt_rep = '20260131'
  AND t.OUT_RUB IS NOT NULL
  AND t.od_flag = 1
  AND t.block_name = 'Привлечение ФЛ'
  AND t.cur = '810';


/* 2) Клиентская витрина под анализ выходов 01–05 февраля (promo / non-promo),
      + базовые суммы по плавающим срочным / НС / ДВС */
DROP TABLE IF EXISTS #client;

SELECT
      dt_rep
    , cli_id

    -- базовые балансы на 31.01
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_floatrate = 1 THEN OUT_RUB ELSE 0 END) AS out_float_term_rub
    , SUM(CASE WHEN SECTION_NAME = N'Накопительный счёт' THEN OUT_RUB ELSE 0 END)          AS out_savings_total_rub
    , SUM(CASE WHEN SECTION_NAME = N'До востребования'  THEN OUT_RUB ELSE 0 END)          AS out_demand_total_rub

    -- PROMO (считаем только по срочным)
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 1 AND dt_close_plan = '2026-02-01' THEN OUT_RUB ELSE 0 END) AS promo_exit_0201_rub
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 1 AND dt_close_plan = '2026-02-02' THEN OUT_RUB ELSE 0 END) AS promo_exit_0202_rub
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 1 AND dt_close_plan = '2026-02-03' THEN OUT_RUB ELSE 0 END) AS promo_exit_0203_rub
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 1 AND dt_close_plan = '2026-02-04' THEN OUT_RUB ELSE 0 END) AS promo_exit_0204_rub
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 1 AND dt_close_plan = '2026-02-05' THEN OUT_RUB ELSE 0 END) AS promo_exit_0205_rub
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 1
               AND (dt_close_plan NOT BETWEEN '2026-02-01' AND '2026-02-05' OR dt_close_plan IS NULL)
               THEN OUT_RUB ELSE 0 END) AS promo_exit_other_rub

    -- NON-PROMO (считаем только по срочным)
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 0 AND dt_close_plan = '2026-02-01' THEN OUT_RUB ELSE 0 END) AS nonpromo_exit_0201_rub
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 0 AND dt_close_plan = '2026-02-02' THEN OUT_RUB ELSE 0 END) AS nonpromo_exit_0202_rub
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 0 AND dt_close_plan = '2026-02-03' THEN OUT_RUB ELSE 0 END) AS nonpromo_exit_0203_rub
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 0 AND dt_close_plan = '2026-02-04' THEN OUT_RUB ELSE 0 END) AS nonpromo_exit_0204_rub
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 0 AND dt_close_plan = '2026-02-05' THEN OUT_RUB ELSE 0 END) AS nonpromo_exit_0205_rub
    , SUM(CASE WHEN SECTION_NAME = N'Срочные ' AND is_promo_3m = 0
               AND (dt_close_plan NOT BETWEEN '2026-02-01' AND '2026-02-05' OR dt_close_plan IS NULL)
               THEN OUT_RUB ELSE 0 END) AS nonpromo_exit_other_rub

INTO #client
FROM #t
GROUP BY dt_rep, cli_id;


/* 3) Сохраняем как постоянную темп-витрину в схему ALM (по аналогии с твоими temp_*) */
DROP TABLE IF EXISTS alm.temp_client_exit_0201_0205;

SELECT *
INTO alm.temp_client_exit_0201_0205
FROM #client;


/* 4) Быстрая проверка: сколько клиентов с любым выходом 01–05 февраля */
SELECT
    COUNT(*) AS clients_cnt
FROM alm.temp_client_exit_0201_0205
WHERE (promo_exit_0201_rub + promo_exit_0202_rub + promo_exit_0203_rub + promo_exit_0204_rub + promo_exit_0205_rub
     + nonpromo_exit_0201_rub + nonpromo_exit_0202_rub + nonpromo_exit_0203_rub + nonpromo_exit_0204_rub + nonpromo_exit_0205_rub) > 0;
