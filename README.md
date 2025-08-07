/*---------------------------------------------------------------*
 | 0. ПАРАМЕТРЫ                                                  |
 *---------------------------------------------------------------*/
USE ALM_TEST;
GO
DECLARE
    @scen        tinyint      = 1,             -- сценарий ключа
    @Anchor      date         = '2025-08-04',  -- последний факт-день
    @HorizonTo   date         = '2025-12-31',  -- конец горизонта
    @BaseRate    decimal(9,4) = 0.0650;        -- базовая ставка

/*---------------------------------------------------------------*
 | 1. ИТОГОВЫЕ ТАБЛИЦЫ                                           |
 *---------------------------------------------------------------*/
IF OBJECT_ID('WORK.Forecast_BalanceDaily_NS','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDaily_NS(
        dt_rep date PRIMARY KEY,
        out_rub_total decimal(20,2),
        rate_avg      decimal(9,4));
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDaily_NS;

IF OBJECT_ID('WORK.NS_Spreads','U') IS NOT NULL
    TRUNCATE TABLE WORK.NS_Spreads;
ELSE
    CREATE TABLE WORK.NS_Spreads(
        anchor date,
        seg    char(1),
        spread decimal(9,4),
        CONSTRAINT PK_NSSpreads PRIMARY KEY(anchor,seg));

IF OBJECT_ID('WORK.NS_PromoRates','U') IS NOT NULL
    TRUNCATE TABLE WORK.NS_PromoRates;
ELSE
    CREATE TABLE WORK.NS_PromoRates(
        month_first date,
        seg         char(1),
        promo_rate  decimal(9,4),
        CONSTRAINT PK_NSPromo PRIMARY KEY(month_first,seg));

/*---------------------------------------------------------------*
 | 2. КАЛЕНДАРЬ                                                  |
 *---------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/*---------------------------------------------------------------*
 | 3. KEY-spot  и  KEY-open                                      |
 *---------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#key_spot') IS NOT NULL DROP TABLE #key_spot;
IF OBJECT_ID('tempdb..#key_open') IS NOT NULL DROP TABLE #key_open;

SELECT fc.DT_REP,fc.KEY_RATE
INTO   #key_spot
FROM   WORK.ForecastKey_Cache_Scen fc
JOIN   #cal c ON c.d = fc.DT_REP
WHERE  fc.Scenario=@scen AND fc.TERM=1;

SELECT DT_REP,TERM,AVG_KEY_RATE
INTO   #key_open
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen
  AND  DT_REP BETWEEN '2000-01-01' AND @HorizonTo;

/*---------------------------------------------------------------*
 | 4. ФАКТ-портфель НС                                           |
 *---------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#bal_fact') IS NOT NULL DROP TABLE #bal_fact;

SELECT  t.con_id,t.cli_id,t.prod_id,
        t.out_rub,t.rate_con,t.is_floatrate,t.termdays,
        CAST(t.dt_open  AS date) AS dt_open,
        CAST(t.dt_close AS date) AS dt_close,
        t.TSEGMENTNAME,
        seg = CASE WHEN t.TSEGMENTNAME=N'Розничный Бизнес' THEN 'R' ELSE 'O' END,
        t.conv
INTO    #bal_fact
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE   t.dt_rep = @Anchor
  AND   t.section_name = N'Накопительный счёт'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.out_rub      IS NOT NULL
  AND  (t.dt_close_fact IS NULL OR t.dt_close_fact >= @Anchor);

/*---------------------------------------------------------------*
 | 5. СПРЕДЫ (фиксируем на @Anchor)                              |
 *---------------------------------------------------------------*/
DECLARE @SpreadO decimal(9,4), @SpreadR decimal(9,4);

SELECT @SpreadO = MAX(CASE WHEN seg='O' THEN rate_con - KEY_RATE END),
       @SpreadR = MAX(CASE WHEN seg='R' THEN rate_con - KEY_RATE END)
FROM (
      SELECT b.seg,b.rate_con,ks.KEY_RATE
      FROM   #bal_fact b
      JOIN   #key_spot ks ON ks.DT_REP=@Anchor
      WHERE  b.prod_id=654 AND DATEDIFF(month,b.dt_open,@Anchor) IN (0,1)
) x;

INSERT INTO WORK.NS_Spreads VALUES (@Anchor,'O',@SpreadO),(@Anchor,'R',@SpreadR);

/*---------------------------------------------------------------*
 | 6. PROMO-окна FIX (654)                                       |
 *---------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#promo') IS NOT NULL DROP TABLE #promo;

SELECT  b.con_id,b.cli_id,b.seg,b.out_rub,
        win_start = b.dt_open,
        win_end   = DATEADD(day,-1,EOMONTH(DATEADD(month,1,b.dt_open))) -- 2 мес –1д
INTO    #promo
FROM    #bal_fact b
WHERE   b.prod_id = 654
  AND   DATEDIFF(month,b.dt_open,@Anchor) IN (0,1);

/*---------------------------------------------------------------*
 | 7. ДНЕВНЫЕ СТАВКИ                                             |
 *---------------------------------------------------------------*/
-- 7.1 FIX-promo + базовый день
IF OBJECT_ID('tempdb..#rate_fixed') IS NOT NULL DROP TABLE #rate_fixed;
SELECT p.con_id,p.cli_id,p.out_rub,
       c.d AS dt_rep,
       rate_con = CASE
                     WHEN c.d < p.win_end
                          THEN (CASE WHEN p.seg='O' THEN @SpreadO ELSE @SpreadR END)
                               + ks.KEY_RATE
                     ELSE @BaseRate
                  END
INTO   #rate_fixed
FROM   #promo p
JOIN   #cal  c  ON c.d BETWEEN p.win_start AND p.win_end
JOIN   #key_spot ks ON ks.DT_REP = c.d;

-- 7.2 FLOAT
IF OBJECT_ID('tempdb..#rate_float') IS NOT NULL DROP TABLE #rate_float;
SELECT f.con_id,f.cli_id,f.out_rub,
       c.d AS dt_rep,
       rate_con = (f.rate_con - ks.KEY_RATE) + ks2.KEY_RATE
INTO   #rate_float
FROM   #bal_fact f
JOIN   #key_spot ks  ON ks.DT_REP = @Anchor          -- фиксируем спред
JOIN   #cal      c  ON c.d BETWEEN f.dt_open AND ISNULL(f.dt_close,@HorizonTo)
JOIN   #key_spot ks2 ON ks2.DT_REP = c.d
WHERE  f.prod_id = 3103;

-- 7.3 FIX-base
IF OBJECT_ID('tempdb..#rate_base') IS NOT NULL DROP TABLE #rate_base;
SELECT b.con_id,b.cli_id,b.out_rub,
       c.d AS dt_rep,
       rate_con = b.rate_con
INTO   #rate_base
FROM   #bal_fact b
JOIN   #cal c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE  b.prod_id = 654
  AND  DATEDIFF(month,b.dt_open,@Anchor) >= 2;

/*---------------------------------------------------------------*
 | 8. ПЕРЕЛИВ 1-го ЧИСЛА (promo-FIX)                             |
 *---------------------------------------------------------------*/
-- 8.1 первый день месяца
IF OBJECT_ID('tempdb..#m1') IS NOT NULL DROP TABLE #m1;
SELECT DISTINCT DATEFROMPARTS(YEAR(d),MONTH(d),1) AS m1 INTO #m1 FROM #cal;

-- 8.2 promo-ставка месяца
IF OBJECT_ID('tempdb..#promo_month') IS NOT NULL DROP TABLE #promo_month;
SELECT  m1 = DATEFROMPARTS(YEAR(ks.DT_REP),MONTH(ks.DT_REP),1),
        seg,
        promo_rate = (CASE WHEN seg='O' THEN @SpreadO ELSE @SpreadR END)
                     + MIN(KEY_RATE)
INTO    #promo_month
FROM    #key_spot  ks
CROSS JOIN (VALUES('O'),('R')) s(seg)
GROUP  BY DATEFROMPARTS(YEAR(ks.DT_REP),MONTH(ks.DT_REP),1), seg;

/* -- FIX: idempotent загрузка promo-ставок ----------------------- */
MERGE WORK.NS_PromoRates AS trg
USING #promo_month AS src
ON   src.m1  = trg.month_first
 AND src.seg = trg.seg
WHEN NOT MATCHED THEN
    INSERT (month_first,seg,promo_rate)
    VALUES (src.m1,src.seg,src.promo_rate)
WHEN MATCHED THEN
    UPDATE SET promo_rate = src.promo_rate;

/*---------------------------------------------------------------*
 | 9. ОБЪЕДИНЯЕМ ВСЕ СТАВКИ                                       |
 *---------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#daily_all') IS NOT NULL DROP TABLE #daily_all;
SELECT con_id,cli_id,out_rub,dt_rep,rate_con
INTO   #daily_all
FROM (
      SELECT con_id,cli_id,out_rub,dt_rep,rate_con FROM #rate_fixed
      UNION ALL
      SELECT con_id,cli_id,out_rub,dt_rep,rate_con FROM #rate_float
      UNION ALL
      SELECT con_id,cli_id,out_rub,dt_rep,rate_con FROM #rate_base
) z;

-- клиенты, у кого есть promo-FIX 1-го числа
;WITH promo_1 AS (
    SELECT DISTINCT cli_id,
           m1 = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
    FROM   #rate_fixed
    WHERE  dt_rep IN (SELECT m1 FROM #m1)
)
, best AS (
    SELECT  f.cli_id,f.dt_rep,f.out_rub,f.rate_con
    FROM    #rate_fixed f
    JOIN    promo_1 p ON p.cli_id = f.cli_id AND p.m1 = f.dt_rep
)
, final AS (
    SELECT con_id = NULL, cli_id, out_rub, dt_rep, rate_con FROM best
    UNION ALL
    SELECT con_id, cli_id, out_rub, dt_rep, rate_con
    FROM   #daily_all d
    WHERE  NOT EXISTS (SELECT 1
                       FROM   promo_1 p
                       WHERE  p.cli_id = d.cli_id
                         AND  p.m1     = DATEFROMPARTS(YEAR(d.dt_rep),MONTH(d.dt_rep),1))
)
INSERT INTO WORK.Forecast_BalanceDaily_NS(dt_rep,out_rub_total,rate_avg)
SELECT  dt_rep,
        SUM(out_rub),
        SUM(out_rub*rate_con)/SUM(out_rub)
FROM    final
GROUP  BY dt_rep;

/*---------------------------------------------------------------*
 | 10. КОНТРОЛЬНЫЙ ВЫВОД                                          |
 *---------------------------------------------------------------*/
PRINT 'rows key_spot  = ' + CAST((SELECT COUNT(*) FROM #key_spot) AS varchar);
PRINT 'rows bal_fact  = ' + CAST((SELECT COUNT(*) FROM #bal_fact) AS varchar);
PRINT 'rows daily_all = ' + CAST((SELECT COUNT(*) FROM #daily_all) AS varchar);

SELECT * FROM WORK.NS_Spreads;
SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,seg;
SELECT * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;
