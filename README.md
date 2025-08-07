/* Диапазон дат, который нужен в задании */
DECLARE @dt_start date = '2025-01-01';
DECLARE @dt_end   date = '2025-07-30';

/*=========================================================
  1. Агрегация по собственной ставке договора (t.rate_con)
=========================================================*/
IF OBJECT_ID('tempdb..#rate_con_agg') IS NOT NULL DROP TABLE #rate_con_agg;

SELECT
    CAST(t.dt_rep AS date)                         AS dt_rep,          -- дата отчёта
    t.section_name,                                                -- «Накопительный счёт»
    t.TSEGMENTNAME,                                                -- сегмент
    t.rate_con                                   AS rate_int,       -- ставка, по которой агрегируем
    SUM(t.out_rub)                               AS sum_out_rub,    -- сумма остатков
    COUNT(DISTINCT t.con_id)                     AS count_con_id    -- количество договоров
INTO #rate_con_agg
FROM alm.ALM.vw_balance_rest_all   t  WITH (NOLOCK)
WHERE t.dt_rep BETWEEN @dt_start AND @dt_end
  AND t.section_name = N'Накопительный счёт'
  AND t.prod_id      = '654'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1          -- без начисленных %
  AND t.cur          = '810'      -- RUB
  AND t.out_rub IS NOT NULL
GROUP BY
    CAST(t.dt_rep AS date),
    t.section_name,
    t.TSEGMENTNAME,
    t.rate_con;

/*=========================================================
  2. Агрегация по ставке из справочника ликвидности (r.rate)
=========================================================*/
IF OBJECT_ID('tempdb..#rate_liq_agg') IS NOT NULL DROP TABLE #rate_liq_agg;

SELECT
    CAST(t.dt_rep AS date)                         AS dt_rep,
    t.section_name,
    t.TSEGMENTNAME,
    r.rate                                      AS rate_int,       -- ставка из Depo­sitContract_Rate
    SUM(t.out_rub)                              AS sum_out_rub,
    COUNT(DISTINCT t.con_id)                    AS count_con_id
INTO #rate_liq_agg
FROM alm.ALM.vw_balance_rest_all           t  WITH (NOLOCK)
LEFT JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = t.con_id
       AND t.dt_rep BETWEEN r.dt_from AND r.dt_to
WHERE t.dt_rep BETWEEN @dt_start AND @dt_end
  AND t.section_name = N'Накопительный счёт'
  AND t.prod_id      = '654'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub IS NOT NULL
GROUP BY
    CAST(t.dt_rep AS date),
    t.section_name,
    t.TSEGMENTNAME,
    r.rate;

/*=========================================================
  3. Быстрый просмотр результатов (по желанию)
=========================================================*/
-- Сводка по собственной ставке
SELECT * 
FROM #rate_con_agg
ORDER BY dt_rep, section_name, TSEGMENTNAME, rate_int;

-- Сводка по ставке ликвидности
SELECT *
FROM #rate_liq_agg
ORDER BY dt_rep, section_name, TSEGMENTNAME, rate_int;

/*---------------------------------------------------------
-- Если нужно свести обе таблицы в один результат:
SELECT COALESCE(c.dt_rep, l.dt_rep)            AS dt_rep,
       COALESCE(c.section_name, l.section_name) AS section_name,
       COALESCE(c.TSEGMENTNAME, l.TSEGMENTNAME) AS TSEGMENTNAME,
       COALESCE(c.rate_int, l.rate_int)         AS rate_int_con,   -- ставка из t
       c.sum_out_rub                            AS sum_out_rub_con,
       c.count_con_id                           AS cnt_con_id_con,
       l.rate_int                               AS rate_int_liq,   -- ставка из r
       l.sum_out_rub                            AS sum_out_rub_liq,
       l.count_con_id                           AS cnt_con_id_liq
FROM #rate_con_agg c
FULL JOIN #rate_liq_agg l
       ON  c.dt_rep        = l.dt_rep
       AND c.section_name  = l.section_name
       AND c.TSEGMENTNAME  = l.TSEGMENTNAME
       AND c.rate_int      = l.rate_int;
---------------------------------------------------------*/
