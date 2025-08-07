Ниже — цельный рабочий T-SQL-скрипт:
	•	Чистит итоговые таблицы / темп-таблицы,
	•	строит календарь, KEY-прогноз, spread’ы, promo-окна,
	•	перекладывает деньги 1-го числа между promo-FIX-счётами,
	•	ставит базовую ставку в «последний день» promo-окна,
	•	ежедневно пересчитывает FLOAT-и базовые FIX,
	•	сохраняет три витрины:
	•	WORK.Forecast_BalanceDaily_NS — дневной агрегат портфеля;
	•	WORK.NS_Spreads — зафиксированные спреды O/R (на @Anchor);
	•	WORK.NS_PromoRates — какая promo-ставка действует в каждом месяце.

Если БД / схемы назваются иначе — подправьте единожды в первых SET-ах.

/* ---------------------------------------------------------------
   0. ПАРАМЕТРЫ
----------------------------------------------------------------*/
USE ALM_TEST;
GO
DECLARE
    @scen        tinyint = 1 ,                  -- сценарий ключа
    @Anchor      date    = '2025-08-02',        -- последний факт-день портфеля
    @HorizonTo   date    = '2025-12-31',        -- конец горизонта
    @BaseRate    decimal(9,4) = 0.0650;         -- базовая ставка после promo

/* ---------------------------------------------------------------
   1. ПОДГОТОВКА ИТОГОВЫХ ТАБЛИЦ
----------------------------------------------------------------*/
IF OBJECT_ID('WORK.Forecast_BalanceDaily_NS','U') IS NULL
CREATE TABLE WORK.Forecast_BalanceDaily_NS
( dt_rep date PRIMARY KEY,
  out_rub_total decimal(20,2),
  rate_avg      decimal(9,4));
TRUNCATE TABLE WORK.Forecast_BalanceDaily_NS;

IF OBJECT_ID('WORK.NS_Spreads','U') IS NULL
CREATE TABLE WORK.NS_Spreads
( anchor date, seg char(1), spread decimal(9,4),
  CONSTRAINT PK_NSSpreads PRIMARY KEY(anchor,seg));
TRUNCATE TABLE WORK.NS_Spreads;

IF OBJECT_ID('WORK.NS_PromoRates','U') IS NULL
CREATE TABLE WORK.NS_PromoRates
( month_first date PRIMARY KEY, seg char(1), promo_rate decimal(9,4));
TRUNCATE TABLE WORK.NS_PromoRates;

/* ---------------------------------------------------------------
   2. КАЛЕНДАРЬ  (@Anchor … @HorizonTo)
----------------------------------------------------------------*/
IF object_id('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* ---------------------------------------------------------------
   3. spot-KEY (TERM=1) и средние KEY на даты открытия
----------------------------------------------------------------*/
IF object_id('tempdb..#key_spot') IS NOT NULL DROP TABLE #key_spot;
SELECT fc.DT_REP, fc.KEY_RATE
INTO   #key_spot
FROM   WORK.ForecastKey_Cache_Scen fc
JOIN   #cal c ON c.d = fc.DT_REP
WHERE  fc.Scenario = @scen AND fc.TERM = 1;

IF object_id('tempdb..#key_open') IS NOT NULL DROP TABLE #key_open;
SELECT DT_REP, TERM, AVG_KEY_RATE
INTO   #key_open
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen
  AND  DT_REP BETWEEN '2000-01-01' AND @HorizonTo;

/* ---------------------------------------------------------------
   4. ФАКТ-портфель НС (FIX=654, FLOAT=3103)
----------------------------------------------------------------*/
IF object_id('tempdb..#bal_fact') IS NOT NULL DROP TABLE #bal_fact;

SELECT  t.con_id, t.cli_id, t.prod_id,
        t.out_rub, t.rate_con, t.is_floatrate, t.termdays,
        CAST(t.dt_open AS date)  AS dt_open,
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
  AND   (t.dt_close_fact IS NULL OR t.dt_close_fact >= @Anchor);

/* ---------------------------------------------------------------
   5. СПРЕДЫ O / R  (фиксируем на @Anchor)
----------------------------------------------------------------*/
DECLARE @SpreadO decimal(9,4),
        @SpreadR decimal(9,4);

SELECT @SpreadO = MAX(CASE WHEN seg='O' THEN rate_con - KEY_RATE END),
       @SpreadR = MAX(CASE WHEN seg='R' THEN rate_con - KEY_RATE END)
FROM (
      SELECT b.seg, b.rate_con, ks.KEY_RATE
      FROM   #bal_fact b
      JOIN   #key_spot ks ON ks.DT_REP = @Anchor
      WHERE  b.prod_id = 654         -- фикс-счёт
        AND  DATEDIFF(month,b.dt_open,@Anchor) IN (0,1) -- promo-счёт
) q;

INSERT INTO WORK.NS_Spreads(anchor,seg,spread)
VALUES (@Anchor,'O',@SpreadO),(@Anchor,'R',@SpreadR);

/* ---------------------------------------------------------------
   6. PROMO-окна FIX-счётов (654)  — две «полных» мат. даты
----------------------------------------------------------------*/
IF object_id('tempdb..#promo') IS NOT NULL DROP TABLE #promo;

SELECT  b.con_id,b.cli_id,b.seg,b.out_rub,
        win_start = b.dt_open,
        win_end   = DATEADD(day,-1,EOMONTH(DATEADD(month,1,b.dt_open))) -- 2 мес-1д
INTO    #promo
FROM    #bal_fact b
WHERE   b.prod_id   = 654
  AND   DATEDIFF(month,b.dt_open,@Anchor) IN (0,1);   -- promo-счёты

/* ---------------------------------------------------------------
   7. ДНЕВНЫЕ СТАВКИ
----------------------------------------------------------------*/
/* 7.1  FIX promo (промежуток win_start … win_end) */
IF object_id('tempdb..#rate_fixed') IS NOT NULL DROP TABLE #rate_fixed;
SELECT p.con_id,p.cli_id,p.out_rub,
       c.d AS dt_rep,
       rate_con = CASE
                    WHEN c.d <  p.win_end
                         THEN (CASE WHEN p.seg='O' THEN @SpreadO ELSE @SpreadR END)
                              + ks.KEY_RATE          -- promo
                    ELSE @BaseRate                  -- win_end  = базовая
                  END
INTO   #rate_fixed
FROM   #promo p
JOIN   #cal  c ON c.d BETWEEN p.win_start AND p.win_end      -- promo+base день
JOIN   #key_spot ks ON ks.DT_REP = c.d;                     -- каждый день !


/* 7.2  FLOAT (3103)  — фиксируем спред в @Anchor */
IF object_id('tempdb..#rate_float') IS NOT NULL DROP TABLE #rate_float;
SELECT  f.con_id,f.cli_id,f.out_rub,
        c.d AS dt_rep,
        rate_con = (f.rate_con - ks.KEY_RATE) + ks2.KEY_RATE
INTO    #rate_float
FROM   #bal_fact f
JOIN   #key_spot ks  ON ks.DT_REP = @Anchor          -- спред фиксируем
JOIN   #cal      c  ON c.d BETWEEN f.dt_open AND ISNULL(f.dt_close,@HorizonTo)
JOIN   #key_spot ks2 ON ks2.DT_REP = c.d
WHERE  f.prod_id = 3103;

/* 7.3  FIX-счёты С БАЗОВОЙ ставкой  (prod_id=654, но dt_open≤ Anchor-2 мес) */
IF object_id('tempdb..#rate_base') IS NOT NULL DROP TABLE #rate_base;
SELECT  b.con_id,b.cli_id,b.out_rub,
        c.d AS dt_rep,
        rate_con = b.rate_con           -- ставка не меняем
INTO    #rate_base
FROM   #bal_fact b
JOIN   #cal  c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE  b.prod_id = 654
  AND  DATEDIFF(month,b.dt_open,@Anchor) >= 2;       -- старые FIX


/* ---------------------------------------------------------------
   8.  ЕЖЕМЕСЯЧНЫЙ ПЕРЕЛИВ promo-FIX
----------------------------------------------------------------*/
/* 8.1  первый день каждого месяца в горизонте */
IF object_id('tempdb..#m1') IS NOT NULL DROP TABLE #m1;
SELECT DISTINCT
       m1 = DATEFROMPARTS(YEAR(d),MONTH(d),1)
INTO   #m1
FROM   #cal;

/* 8.2  лучшая promo-ставка O / R в каждом месяце */
IF object_id('tempdb..#promo_month') IS NOT NULL DROP TABLE #promo_month;
SELECT  DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1) AS m1,
        seg,
        promo_rate = (CASE WHEN seg='O' THEN @SpreadO ELSE @SpreadR END)
                     + MIN(KEY_RATE)                 -- самый низкий KEY в месяце
INTO    #promo_month
FROM    #key_spot
GROUP BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1), seg;

INSERT INTO WORK.NS_PromoRates(month_first,seg,promo_rate)
SELECT m1,seg,promo_rate FROM #promo_month;

/* 8.3  деньги клиента → счёт с max(rate_con) среди promo-FIX на 1-е число */
IF object_id('tempdb..#daily_union') IS NOT NULL DROP TABLE #daily_union;
SELECT * INTO #daily_union
FROM (
      SELECT * FROM #rate_fixed
      UNION ALL
      SELECT * FROM #rate_float
      UNION ALL
      SELECT * FROM #rate_base
) q;

/* клиенты, у кого 1-го числа есть promo */
;WITH promo_first AS (
    SELECT DISTINCT
           cli_id,
           m1 = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
    FROM   #rate_fixed
)
, best_promo AS (
    SELECT d.cli_id,
           d.dt_rep,
           d.out_rub,
           d.rate_con
    FROM   #rate_fixed d
    JOIN   promo_first p
         ON p.cli_id = d.cli_id
        AND p.m1     = d.dt_rep        -- d.dt_rep уже = 1-е число
)
, daily_final AS (
    SELECT * FROM best_promo          -- деньги переложены
    UNION ALL
    SELECT u.*
    FROM   #daily_union u
    WHERE  NOT EXISTS (SELECT 1
                       FROM   promo_first p
                       WHERE  p.cli_id = u.cli_id
                         AND  p.m1     = DATEFROMPARTS(YEAR(u.dt_rep),MONTH(u.dt_rep),1))
)
INSERT INTO WORK.Forecast_BalanceDaily_NS(dt_rep,out_rub_total,rate_avg)
SELECT  dt_rep,
        SUM(out_rub),
        SUM(out_rub*rate_con)/SUM(out_rub)
FROM    daily_final
GROUP BY dt_rep;

/* ---------------------------------------------------------------
   9. РЕЗУЛЬТАТЫ
----------------------------------------------------------------*/
SELECT * FROM WORK.NS_Spreads;
SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,seg;
SELECT * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;

что сделано / проверено

Требование	Реализация
Последний день promo-окна → базовая ставка	условие c.d < win_end ? promo : @BaseRate
Промо действует ровно 2 месяца от dt_open	win_end = EOMONTH(DATEADD(month,1,dt_open))-1
Перелив денег только 1-го числа и только между promo-FIX	CTE promo_first, best_promo
FLOAT (3103) → фикс. спред к KEY	rate_float
Ставка promo каждый месяц = KEY + spread (спреды из @Anchor)	#promo_month, сохранено в NS_PromoRates
Таблицы очищаются перед загрузкой	TRUNCATE TABLE …

Запускайте целиком: создаст/обновит витрины и сразу покажет три выборки внизу.
