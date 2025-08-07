ниже — цельный скрипт, который
	•	генерирует три ежедневные агрег-ленты
(FLOAT / FIX-base / FIX-promo);
	•	пишет их в отдельные рабочие таблицы
WORK.Forecast_NS_Float,
WORK.Forecast_NS_FixBase,
WORK.Forecast_NS_Promo;
	•	затем суммирует их в
WORK.Forecast_BalanceDaily_NS.

так у вас всегда есть точные подпорки для отладки-сверки:
любой “разбухший” объём сразу видно, из какой подсекции он пришёл.

/* ═══════════════════════════════════════════════════════════════
   NS forecast (FLOAT | FIX-base | FIX-promo 2-мес roll)
   v.2025-08-07 split-agg
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/* ── 0. служебный календарь + KEY ------------------------- */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal)<@HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

DROP TABLE IF EXISTS #key;
SELECT DT_REP,KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

DECLARE @KeyAnchor decimal(9,4) =
        (SELECT KEY_RATE FROM #key WHERE DT_REP=@Anchor);

/* ── 1. портфель @Anchor --------------------------------- */
DROP TABLE IF EXISTS #bal;
SELECT  t.con_id ,t.cli_id ,t.prod_id ,
        t.out_rub,t.rate_con,t.is_floatrate,
        CAST(t.dt_open AS date)  dt_open ,
        CAST(t.dt_close AS date) dt_close ,
        t.TSEGMENTNAME
INTO    #bal
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE   t.dt_rep=@Anchor
  AND   t.section_name=N'Накопительный счёт'
  AND   t.block_name  =N'Привлечение ФЛ'
  AND   t.od_flag=1 AND t.cur='810'
  AND   t.out_rub IS NOT NULL
  AND  (t.dt_close_fact IS NULL OR t.dt_close_fact>=@Anchor);

/* =================================================================
   2.  FLOAT  (3103)  — спред фиксируем в dt_open
================================================================= */
DROP TABLE IF EXISTS #FLOAT_daily;
SELECT  b.con_id ,b.cli_id ,b.TSEGMENTNAME ,b.out_rub,
        c.d dt_rep,
        (b.rate_con-k0.KEY_RATE)+k1.KEY_RATE AS rate_con
INTO    #FLOAT_daily
FROM    #bal b
JOIN    #key k0 ON k0.DT_REP=b.dt_open
JOIN    #cal c  ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
JOIN    #key k1 ON k1.DT_REP=c.d
WHERE   b.prod_id=3103;

/* агрегируем в рабочую таблицу */
DROP TABLE IF EXISTS WORK.Forecast_NS_Float;
SELECT  dt_rep,
        SUM(out_rub)                       out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) rate_avg
INTO    WORK.Forecast_NS_Float
FROM   #FLOAT_daily
GROUP  BY dt_rep;

/* =================================================================
   3.  FIX-base 654  (dt_open ≤ июнь-25) — ставка константа
================================================================= */
DROP TABLE IF EXISTS #FIX_base_daily;
SELECT  b.con_id ,b.cli_id ,b.TSEGMENTNAME ,b.out_rub,
        c.d dt_rep,
        b.rate_con AS rate_con
INTO    #FIX_base_daily
FROM    #bal b
JOIN    #cal c ON c.d BETWEEN b.dt_open AND @HorizonTo
WHERE   b.prod_id=654 AND b.dt_open<'2025-07-01';

DROP TABLE IF EXISTS WORK.Forecast_NS_FixBase;
SELECT  dt_rep,
        SUM(out_rub)                       out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) rate_avg
INTO    WORK.Forecast_NS_FixBase
FROM   #FIX_base_daily
GROUP  BY dt_rep;

/* =================================================================
   4.  FIX-promo (июль / август-25 c 2-мес. rollover)
================================================================= */
------------------------------------------------------------------
/* 4-1. спреды сегментов фиксируем на @Anchor                     */
DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

WITH aug AS (
     SELECT TSEGMENTNAME,
            SUM(out_rub*rate_con)/SUM(out_rub) AS w_rate
     FROM   #bal
     WHERE  prod_id=654
       AND  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
     GROUP  BY TSEGMENTNAME)
SELECT @Spread_DChbo  = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО'            THEN w_rate END)-@KeyAnchor,
       @Spread_Retail = MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END)-@KeyAnchor
FROM   aug;

/* 4-2. рекурсивные окна                                              */
DROP TABLE IF EXISTS #promo_win;
;WITH seed AS (
      SELECT b.con_id ,b.cli_id ,b.TSEGMENTNAME ,b.out_rub,
             win_start=b.dt_open,
             win_end  =EOMONTH(b.dt_open,1)
      FROM   #bal b
      WHERE  b.prod_id=654
        AND  b.dt_open BETWEEN '2025-07-01' AND '2025-08-31')
, seq AS (
      SELECT *,0 n FROM seed
      UNION ALL
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,
             DATEADD(day,1,win_end),
             EOMONTH(DATEADD(day,1,win_end),1),
             n+1
      FROM   seq
      WHERE  DATEADD(day,1,win_end)<=@HorizonTo)
SELECT * INTO #promo_win FROM seq OPTION (MAXRECURSION 0);

/* 4-3. дневная сетка promo (+ базовый день)                          */
DROP TABLE IF EXISTS #FIX_promo_daily;
SELECT  p.con_id ,p.cli_id ,p.TSEGMENTNAME ,p.out_rub,
        c.d dt_rep,
        CASE
             WHEN c.d<=p.win_end
                  THEN CASE p.TSEGMENTNAME
                           WHEN N'ДЧБО'            THEN @Spread_DChbo  + k.KEY_RATE
                           ELSE                         @Spread_Retail + k.KEY_RATE
                       END
             WHEN c.d=DATEADD(day,1,p.win_end)
                  THEN @BaseRate
        END AS rate_con
INTO    #FIX_promo_daily
FROM    #promo_win p
JOIN    #cal c ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN    #key k ON k.DT_REP=c.d;

/* 4-4. перелив promo-денег 1-го числа                               */
DROP TABLE IF EXISTS #p1;
SELECT DISTINCT cli_id,
       DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1) m1
INTO   #p1
FROM   #FIX_promo_daily
WHERE  dt_rep=DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1);

DROP TABLE IF EXISTS #p1_glue;
SELECT  NULL             con_id,
        p.cli_id,
        NULL             TSEGMENTNAME,
        SUM(f.out_rub)   out_rub,
        p.m1             dt_rep,
        MAX(f.rate_con)  rate_con
INTO    #p1_glue
FROM    #p1 p
JOIN    #FIX_promo_daily f
       ON f.cli_id=p.cli_id AND f.dt_rep=p.m1
GROUP  BY p.cli_id,p.m1;

DROP TABLE IF EXISTS #p_not1;
SELECT * INTO #p_not1
FROM   #FIX_promo_daily
WHERE  dt_rep<>DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1);

/* итог promo-агрегат */
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;
SELECT  dt_rep,
        SUM(out_rub)                       out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) rate_avg
INTO    WORK.Forecast_NS_Promo
FROM (
      SELECT * FROM #p_not1
      UNION ALL
      SELECT * FROM #p1_glue) z
GROUP  BY dt_rep;

/* =================================================================
   5. Финальный портфель  (сумма трёх агрег-лент)
================================================================= */
DROP TABLE IF EXISTS WORK.Forecast_BalanceDaily_NS;
CREATE TABLE WORK.Forecast_BalanceDaily_NS(
  dt_rep date PRIMARY KEY,
  out_rub_total decimal(20,2),
  rate_avg      decimal(9,4));

INSERT WORK.Forecast_BalanceDaily_NS
SELECT  f.dt_rep,
        f.out_rub_total + b.out_rub_total + p.out_rub_total           AS out_rub_total,
        /* средневзв. по сумме трёх категорий */
        (f.out_rub_total*f.rate_avg +
         b.out_rub_total*b.rate_avg +
         p.out_rub_total*p.rate_avg)
/        NULLIF(f.out_rub_total+b.out_rub_total+p.out_rub_total,0)     AS rate_avg
FROM   WORK.Forecast_NS_Float   f
JOIN   WORK.Forecast_NS_FixBase b ON b.dt_rep=f.dt_rep
JOIN   WORK.Forecast_NS_Promo   p ON p.dt_rep=f.dt_rep;

/* =================================================================
   6. DEBUG-вывод
================================================================= */
PRINT N'╔ sample «ступеньки» ════════════════════════════════════';
SELECT dt_rep,out_rub_total,rate_avg
FROM   WORK.Forecast_BalanceDaily_NS
WHERE  dt_rep IN ('2025-08-30','2025-08-31',
                  '2025-09-01','2025-09-15',
                  '2025-09-30','2025-10-01')
ORDER  BY dt_rep;
PRINT N'╚════════════════════════════════════════════════════════';

PRINT N'=== spread (модель) ===';
SELECT N'ДЧБО' AS seg, @Spread_DChbo  AS spread  UNION ALL
SELECT N'Розничный бизнес', @Spread_Retail;

PRINT N'=== promo-rate by month ===';
WITH kmin AS (
     SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) m1,
            MIN(KEY_RATE) kmin
     FROM   #key
     GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
SELECT m1,
       N'ДЧБО'              AS seg,
       kmin+@Spread_DChbo   AS promo_rate
FROM kmin
UNION ALL
SELECT m1,N'Розничный бизнес',kmin+@Spread_Retail FROM kmin
ORDER BY m1,seg;

PRINT N'=== FLOAT TOP-10 ===';   SELECT TOP 10 * FROM WORK.Forecast_NS_Float   ORDER BY dt_rep;
PRINT N'=== FIX-base TOP-10 ===';SELECT TOP 10 * FROM WORK.Forecast_NS_FixBase ORDER BY dt_rep;
PRINT N'=== PROMO TOP-10 ===';   SELECT TOP 10 * FROM WORK.Forecast_NS_Promo   ORDER BY dt_rep;
PRINT N'=== portfolio TOP-40 ===';
SELECT TOP 40 * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;
GO

Все три промежуточные агрегаты (WORK.Forecast_NS_*) теперь
хранят ежедневные тоталы и средневзвешенные ставки, а итоговый
портфель — просто их сумма.
Так гарантированно не появится лишних объёмов при соединении потоков,
и удобно проверять каждую категорию отдельно.
