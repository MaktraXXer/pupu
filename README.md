/* ═══════════════════════════════════════════════════════════════
   NS-forecast  — ЧАСТЬ 1  (ТОЛЬКО   FLOAT   и   FIX-base)
   * FLOAT     → prod_id 3103, спред фиксируем на dt_open
   * FIX-base  → prod_id 654,  dt_open < 2025-07-01, ставка константа
   создаёт / перезаписывает:
       #cal, #key, #bal
       #FLOAT_daily            + WORK.Forecast_NS_Float
       #FIX_base_daily         + WORK.Forecast_NS_FixBase
   v.2025-08-08
════════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31';

/* ───────────── 1. календарь ───────────── */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* ───────────── 2. KEY-spot (TERM=1) ───── */
IF OBJECT_ID('tempdb..#key') IS NOT NULL DROP TABLE #key;
SELECT DT_REP,KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* ───────────── 3. факт-портфель @Anchor ─ */
IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;
SELECT  t.con_id,t.cli_id,t.prod_id,
        t.out_rub,t.rate_con,
        CAST(t.dt_open  AS date) dt_open,
        CAST(t.dt_close AS date) dt_close,
        t.TSEGMENTNAME
INTO    #bal
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE   t.dt_rep=@Anchor
  AND   t.section_name=N'Накопительный счёт'
  AND   t.block_name  =N'Привлечение ФЛ'
  AND   t.od_flag=1 AND t.cur='810'
  AND   t.out_rub IS NOT NULL
  AND  (t.dt_close_fact IS NULL OR t.dt_close_fact>=@Anchor);

/* ───────────── 4-A.  FLOAT daily ──────── */
IF OBJECT_ID('tempdb..#FLOAT_daily') IS NOT NULL DROP TABLE #FLOAT_daily;
SELECT  b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
        c.d                                      AS dt_rep,
        (b.rate_con-k0.KEY_RATE)+k1.KEY_RATE     AS rate_con
INTO    #FLOAT_daily
FROM    #bal b
JOIN    #key k0 ON k0.DT_REP = b.dt_open       -- фиксируем спред
JOIN    #cal c  ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
JOIN    #key k1 ON k1.DT_REP = c.d
WHERE   b.prod_id = 3103;

/* агрегат FLOAT */
IF OBJECT_ID('WORK.Forecast_NS_Float','U') IS NOT NULL
    TRUNCATE TABLE WORK.Forecast_NS_Float
ELSE
    CREATE TABLE WORK.Forecast_NS_Float(
        dt_rep date PRIMARY KEY,
        out_rub_total  decimal(20,2),
        rate_avg       decimal(9,4));

INSERT WORK.Forecast_NS_Float
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #FLOAT_daily
GROUP  BY dt_rep;

/* ───────────── 4-B.  FIX-base daily ──── */
IF OBJECT_ID('tempdb..#FIX_base_daily') IS NOT NULL DROP TABLE #FIX_base_daily;
SELECT  b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
        c.d            AS dt_rep,
        b.rate_con     AS rate_con
INTO    #FIX_base_daily
FROM    #bal b
JOIN    #cal c ON c.d BETWEEN b.dt_open AND @HorizonTo
WHERE   b.prod_id = 654
  AND   b.dt_open < '2025-07-01';

/* агрегат FIX-base */
IF OBJECT_ID('WORK.Forecast_NS_FixBase','U') IS NOT NULL
    TRUNCATE TABLE WORK.Forecast_NS_FixBase
ELSE
    CREATE TABLE WORK.Forecast_NS_FixBase(
        dt_rep date PRIMARY KEY,
        out_rub_total  decimal(20,2),
        rate_avg       decimal(9,4));

INSERT WORK.Forecast_NS_FixBase
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #FIX_base_daily
GROUP  BY dt_rep;

/* ───────────── 5. вывод для проверки ─── */
PRINT N'==== FLOAT daily (TOP-30) ====';
SELECT TOP (30) * FROM #FLOAT_daily ORDER BY dt_rep,con_id;

PRINT N'==== FIX-base daily (TOP-30) ====';
SELECT TOP (30) * FROM #FIX_base_daily ORDER BY dt_rep,con_id;

PRINT N'==== агрегаты (первые 10 дней) ====';
SELECT 'FLOAT'   AS bucket,* FROM WORK.Forecast_NS_Float   ORDER BY dt_rep
UNION ALL
SELECT 'FIX-base',*            FROM WORK.Forecast_NS_FixBase ORDER BY dt_rep;
GO
