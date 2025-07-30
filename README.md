Ниже ― цельный T-SQL-скрипт, который:
	•	строит прогноз только для одного сценария (по умолчанию – № 1, можно поменять параметр @scen);
	•	использует единственный набор клиентских ставок “УЧК / O” и постоянный Δ(O-R);
	•	конвертирует «с конечной → ежемесячную» ставку для любого срока, а не только 46 – 196 дней;
	•	перед расчётом очищает/создаёт все рабочие таблицы.

/*====================================================================
     ПАРАМЕТРЫ
  ====================================================================*/
DECLARE @scen        tinyint = 1 ;           -- какой сценарий ЦБ берём
DECLARE @Anchor      date    = '2025-07-28'; -- последний факт-день портфеля
DECLARE @HorizonTo   date    = '2025-12-31'; -- конец прогноза           

/*====================================================================
     0.  СПРАВОЧНИКИ «TO-BE»
  ====================================================================*/
IF OBJECT_ID('tempdb..#to_be_O') IS NOT NULL DROP TABLE #to_be_O;
IF OBJECT_ID('tempdb..#delta_R') IS NOT NULL DROP TABLE #delta_R;

/* ——— клиент-O (УЧК) ——— */
CREATE TABLE #to_be_O
( term_nom int PRIMARY KEY,         -- 61, 91, 122 …
  o_rate   decimal(9,4) );          -- уже в долях (17 % ⇒ 0.1700)

INSERT #to_be_O VALUES
 (  61,0.1770),(  91,0.1780),( 122,0.1710),
 ( 181,0.1660),( 274,0.1600),( 365,0.1560),
 ( 548,0.1450),( 730,0.1430),(1100,0.1380);

/* ——— Δ(O-R) ——— */
CREATE TABLE #delta_R
( term_nom int PRIMARY KEY,
  delta_ro decimal(9,4));

INSERT #delta_R VALUES
 (  61,0.0040),(  91,0.0040),( 122,0.0050),
 ( 181,0.0040),( 274,0.0040),( 365,0.0020),
 ( 548,0.0070),( 730,0.0070),(1100,0.0070);

/* диапазоны сроков (для мэпинга к “номиналу”) */
IF OBJECT_ID('tempdb..#term_rng') IS NOT NULL DROP TABLE #term_rng;
CREATE TABLE #term_rng(term_nom int PRIMARY KEY, term_lo int, term_hi int);
INSERT #term_rng VALUES
 ( 61,  46,  76),( 91,  76, 106),(122, 107, 137),
 (181, 166, 196),(274, 259, 289),(365, 352, 382),
 (548, 535, 565),(730, 715, 745),(1100,1085,1115);

/*====================================================================
     1.  КАЛЕНДАРЬ  и  spot-KEY (= TERM=1) по выбранному сценарию
  ====================================================================*/
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

IF OBJECT_ID('tempdb..#key_spot') IS NOT NULL DROP TABLE #key_spot;
SELECT fc.DT_REP,fc.KEY_RATE
INTO   #key_spot
FROM   WORK.ForecastKey_Cache_Scen fc
JOIN   #cal c ON c.d=fc.DT_REP
WHERE  fc.Scenario=@scen AND fc.TERM=1;

/* средние KEY-rates на все возможные даты открытия */
IF OBJECT_ID('tempdb..#key_open') IS NOT NULL DROP TABLE #key_open;
SELECT DT_REP,TERM,AVG_KEY_RATE
INTO   #key_open
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen
  AND  DT_REP BETWEEN '2000-01-01' AND @HorizonTo;

/*====================================================================
     2.  ФАКТ-портфель на @Anchor (FIX и FLOAT вместе)
  ====================================================================*/
IF OBJECT_ID('tempdb..#bal_fact') IS NOT NULL DROP TABLE #bal_fact;

SELECT  t.con_id,t.out_rub,t.rate_con,t.is_floatrate,t.termdays,
        t.dt_open,t.dt_close,t.TSEGMENTNAME,t.conv
INTO    #bal_fact
FROM    ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE   t.dt_rep=@Anchor
  AND   t.section_name=N'Срочные'
  AND   t.block_name=N'Привлечение ФЛ'
  AND   t.od_flag=1
  AND   t.cur='810'
  AND   t.out_rub IS NOT NULL;

/*====================================================================
     3.  REF-SPREAD ДЛЯ FIX
  ====================================================================*/
IF OBJECT_ID('tempdb..#ref') IS NOT NULL DROP TABLE #ref;

SELECT s.seg,                       /* 'R' | 'O' */
       tr.term_nom,tr.term_lo,tr.term_hi,
       tobe_end = o.o_rate,
       key_avg  = k.AVG_KEY_RATE,
       delta_ro = d.delta_ro
INTO   #ref
FROM  (VALUES('R'),('O'))           AS s(seg)
JOIN  #term_rng  tr                ON 1=1
JOIN  #to_be_O   o                 ON o.term_nom=tr.term_nom
JOIN  #delta_R   d                 ON d.term_nom=tr.term_nom
JOIN  #key_open  k                 ON k.DT_REP=@Anchor AND k.TERM=tr.term_nom;

/* Spread «конец» и «ежемесячная» (для любого срока!) */
ALTER TABLE #ref ADD
    spread_end     decimal(9,6),
    spread_monthly decimal(9,6);

UPDATE #ref
SET spread_end =
        CASE WHEN seg='O' THEN tobe_end-key_avg
             ELSE tobe_end-delta_ro-key_avg END,
    spread_monthly =
        /* fnc_IntRate: пересчёт под monthly-quoting */
        CAST([LIQUIDITY].[liq].[fnc_IntRate]
                ( CASE WHEN seg='O' THEN tobe_end
                       ELSE tobe_end-delta_ro END,
                  'at the end','monthly',term_nom,1)
             AS decimal(9,6)) - key_avg;

/*====================================================================
     4.  БАЗА с spread_final (FIX)  и  spread_float (FLOAT)
  ====================================================================*/
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;

;WITH m AS (
    SELECT  b.con_id,b.out_rub,b.rate_con,b.is_floatrate,b.termdays,
            b.dt_open,b.dt_close,
            seg   = CASE WHEN b.TSEGMENTNAME=N'Розничный Бизнес' THEN 'R' ELSE 'O' END,
            conv  = b.conv,
            spread_float = b.rate_con - ks.KEY_RATE,
            key_avg_open = k.AVG_KEY_RATE
    FROM    #bal_fact  b
    LEFT    JOIN #key_spot ks ON ks.DT_REP=@Anchor      -- spot = Anchor
    LEFT    JOIN #key_open k  ON k.DT_REP=b.dt_open AND k.TERM=b.termdays
)
SELECT  m.con_id,m.out_rub,m.is_floatrate,m.termdays,
        m.dt_open,m.dt_close,m.spread_float,
        spread_fix_fact = m.rate_con - m.key_avg_open,
        spread_final =
             CASE WHEN m.is_floatrate=1 THEN NULL
                  ELSE COALESCE(                       /* справочник */
                           CASE WHEN m.conv='AT_THE_END'
                                    THEN r.spread_end
                                    ELSE r.spread_monthly
                           END,
                           m.rate_con - m.key_avg_open) /* fallback */
             END
INTO    #base
FROM    m
LEFT    JOIN #ref r
       ON r.seg=m.seg
      AND m.termdays BETWEEN r.term_lo AND r.term_hi;

/*====================================================================
     5.  ROLL-OVER-цепочки (n = 0,1,2,…)
  ====================================================================*/
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;
;WITH seq AS (
    SELECT con_id,out_rub,is_floatrate,termdays,dt_open,
           spread_float,spread_fix = spread_fix_fact,spread_final,n=0
    FROM   #base
    UNION ALL
    SELECT s.con_id,s.out_rub,s.is_floatrate,s.termdays,
           DATEADD(day,s.termdays,s.dt_open),
           s.spread_float,s.spread_final,s.spread_final,n+1
    FROM   seq s
    WHERE  DATEADD(day,s.termdays,s.dt_open) <= @HorizonTo
)
SELECT con_id,out_rub,is_floatrate,termdays,
       dt_open,dt_close = DATEADD(day,termdays,dt_open),
       spread_float,spread_fix
INTO   #rolls
FROM   seq OPTION (MAXRECURSION 0);

/*====================================================================
     6.  ПОСУТОЧНЫЕ СТАВКИ и АГРЕГАТ
  ====================================================================*/
IF OBJECT_ID('tempdb..#daily') IS NOT NULL DROP TABLE #daily;
SELECT c.d AS dt_rep,
       r.con_id,
       r.out_rub,
       rate_con = CASE
                     WHEN r.is_floatrate=1
                          THEN ks.KEY_RATE + r.spread_float
                     ELSE ISNULL(fk.AVG_KEY_RATE + r.spread_fix,
                                 r.spread_fix)
                  END
INTO   #daily
FROM   #cal   c
JOIN   #rolls r ON c.d BETWEEN r.dt_open AND DATEADD(day,-1,r.dt_close)
LEFT  JOIN #key_spot ks ON ks.DT_REP=c.d
LEFT  JOIN #key_open fk ON fk.DT_REP=r.dt_open AND fk.TERM=r.termdays;

/*====================================================================
     7.  ЗАПИСЫВАЕМ РЕЗУЛЬТАТ
  ====================================================================*/
IF OBJECT_ID('WORK.Forecast_BalanceDaily_cases','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDaily_cases
    ( case_cd varchar(5),dt_rep date,
      out_rub_total decimal(20,2),rate_con decimal(9,4),
      CONSTRAINT PK_FBD_cases PRIMARY KEY(case_cd,dt_rep));
TRUNCATE TABLE WORK.Forecast_BalanceDaily_cases;

INSERT INTO WORK.Forecast_BalanceDaily_cases(case_cd,dt_rep,out_rub_total,rate_con)
SELECT '1O',                /* код кейса: сценарий 1, set O */
       dt_rep,
       SUM(out_rub),
       SUM(out_rub*rate_con)/SUM(out_rub)
FROM   #daily
GROUP  BY dt_rep;

/*====================================================================
     ГОТОВО
  ====================================================================*/
SELECT * FROM WORK.Forecast_BalanceDaily_cases ORDER BY dt_rep;

что изменилось / почему это работает

Блок	Изменение	Комментарий
0	#to_be_O, #delta_R — только один набор О-ставок + Δ(O-R)	ставки и спреды взяты из вашей таблицы (уже в долях)
3	spread_monthly считается для любого term_nom, условие BETWEEN 46 AND 196 убрано	функция liq.fnc_IntRate работает со всеми сроками
4	spread_final для FIX берётся из справочника; если депозит c ежемес. купоном → spread_monthly, иначе spread_end	покрывает все сроки
5-7	убран цикл по кейсам: формируется один кейс '1O'	только выбранный сценарий

скрипт полностью автономен: он чистит/создаёт временные и рабочие таблицы, поэтому можно запускать столько раз, сколько нужно — данные будут пересчитаны «с нуля».
