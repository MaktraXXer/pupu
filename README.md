/* ═══════════════════════════════════════════════════════════════
   NS-forecast :  ЧАСТЬ 1  (ТОЛЬКО FLOAT и FIX-base)
     – FLOAT  : prod_id 3103,   спред фиксируем в dt_open
     – FIX-base: prod_id 654,   dt_open < 2025-07-01, ставка константа
   создаёт / перезаписывает:
       #cal, #key, #bal
       #FLOAT_daily,   WORK.Forecast_NS_Float
       #FIX_base_daily,WORK.Forecast_NS_FixBase
   вся база — ALM_TEST
   v.2025-08-08
════════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31';

/* ───────────────────────── 1. календарь горизонта ─────────── */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* ───────────────────────── 2. ключевая ставка spot ─────────── */
DROP TABLE IF EXISTS #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* ───────────────────────── 3. факт-портфель @Anchor ───────── */
DROP TABLE IF EXISTS #bal;
SELECT  t.con_id,t.cli_id,t.prod_id,
        t.out_rub,t.rate_con,
        CAST(t.dt_open  AS date) AS dt_open,
        CAST(t.dt_close AS date) AS dt_close,
        t.TSEGMENTNAME
INTO    #bal
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE   t.dt_rep = @Anchor
  AND   t.section_name = N'Накопительный счёт'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.out_rub IS NOT NULL
  AND  (t.dt_close_fact IS NULL OR t.dt_close_fact >= @Anchor);

/* ───────────────────────── 4-A. FLOAT  (spred fix @open) ───── */
DROP TABLE IF EXISTS #FLOAT_daily;
SELECT  b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
        c.d                              AS dt_rep,
        (b.rate_con-k0.KEY_RATE) + k1.KEY_RATE AS rate_con
INTO    #FLOAT_daily
FROM    #bal b
JOIN    #key k0 ON k0.DT_REP = b.dt_open   -- фиксируем спред
JOIN    #cal c  ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
JOIN    #key k1 ON k1.DT_REP = c.d
WHERE   b.prod_id = 3103;

/* агрегируем на один день */
TRUNCATE TABLE IF EXISTS WORK.Forecast_NS_Float;
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
INTO    WORK.Forecast_NS_Float
FROM   #FLOAT_daily
GROUP  BY dt_rep;

/* ───────────────────────── 4-B. FIX-base  (константа) ─────── */
DROP TABLE IF EXISTS #FIX_base_daily;
SELECT  b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
        c.d           AS dt_rep,
        b.rate_con    AS rate_con
INTO    #FIX_base_daily
FROM    #bal b
JOIN    #cal c ON c.d BETWEEN b.dt_open AND @HorizonTo
WHERE   b.prod_id = 654
  AND   b.dt_open < '2025-07-01';

TRUNCATE TABLE IF EXISTS WORK.Forecast_NS_FixBase;
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
INTO    WORK.Forecast_NS_FixBase
FROM   #FIX_base_daily
GROUP  BY dt_rep;

/* ───────────────────────── 5. контрольный вывод ───────────── */
PRINT N'==== FLOAT (все дни, все con_id) ====';  
SELECT * FROM #FLOAT_daily ORDER BY dt_rep, con_id;

PRINT N'==== FIX-base (все дни, все con_id) ====';  
SELECT * FROM #FIX_base_daily ORDER BY dt_rep, con_id;

PRINT N'==== агрегаты ====';  
SELECT 'FLOAT'   AS bucket,* FROM WORK.Forecast_NS_Float
UNION ALL
SELECT 'FIX-base',*             FROM WORK.Forecast_NS_FixBase
ORDER BY dt_rep;
