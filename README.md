/*---------------------------------------------------------------------- 
  ПАРАМЕТРЫ ВЫЧИСЛЕНИЯ 
----------------------------------------------------------------------*/ 
DECLARE @Anchor      date = '2025-07-16',      -- последний факт 
        @HorizonTo   date = '2025-09-30';      -- конец прогноза 

/*---------------------------------------------------------------------- 
  INPUT:  TO-BE-O RATES  (два набора — A и B)  + Δ(O-R) по каждому сроку 
----------------------------------------------------------------------*/ 
IF OBJECT_ID('tempdb..#to_be_O'      ) IS NOT NULL DROP TABLE #to_be_O; 
IF OBJECT_ID('tempdb..#delta_R'      ) IS NOT NULL DROP TABLE #delta_R; 
IF OBJECT_ID('tempdb..#cases'        ) IS NOT NULL DROP TABLE #cases; 

CREATE TABLE #to_be_O                   /* один набор = одна строка */ 
( set_cd  char(1),                      -- A | B 
  term_nom int,                         -- 61, 91, 122 … 
  o_rate   decimal(9,4) );              -- ПРОЦЕНТ *в долях*: 17 % ⇒ 0.1700 

/* ―― set A  (-3 п.п. в июле, -2 п.п. в сентябре) ―― */ 
INSERT #to_be_O VALUES 
('A',61,0.1740),('A',91,0.1700),('A',122,0.1670),('A',181,0.1610), 
('A',274,0.1560),('A',365,0.1540),('A',548,0.1430),('A',730,0.1410),('A',1100,0.1360), 
/* ―― set B  (-2 п.п. в июле, -2 п.п. в сентябре) ―― */ 
('B',61,0.1840),('B',91,0.1800),('B',122,0.1750),('B',181,0.1660), 
('B',274,0.1590),('B',365,0.1570),('B',548,0.1460),('B',730,0.1440),('B',1100,0.1390); 

/* Δ(O-R) одинакова для set A и B */ 
CREATE TABLE #delta_R(term_nom int PRIMARY KEY, delta_ro decimal(9,4)); 
INSERT #delta_R VALUES 
(61,0.0050),(91,0.0040),(122,0.0060),(181,0.0040), 
(274,0.0040),(365,0.0020),(548,0.0070),(730,0.0070),(1100,0.0070); 

/* какие кейсы вычисляем:  <сценарий><набор O-ставок> */ 
CREATE TABLE #cases 
( case_cd varchar(5) PRIMARY KEY,  -- напр. 1A, 2A, 3B 
  scen    tinyint,                -- сценарий ЦБ (1/2/3) 
  set_cd  char(1) );              -- A | B 

INSERT #cases VALUES ('3B',3,'B'), ('1A',1,'A'), ('2A',2,'A'); 

/*---------------------------------------------------------------------- 
  0. календарь и spot-KEY (формируется позднее по сценарию) 
----------------------------------------------------------------------*/ 
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal; 
SELECT d=@Anchor INTO #cal; 
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo 
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal; 

/*---------------------------------------------------------------------- 
  1. факт-портфель на @Anchor  (FIX + FLOAT вместе) 
----------------------------------------------------------------------*/ 
IF OBJECT_ID('tempdb..#bal_fact') IS NOT NULL DROP TABLE #bal_fact; 

SELECT  t.con_id, t.out_rub, t.rate_con, t.is_floatrate, 
        t.termdays, t.dt_open, t.dt_close, t.TSEGMENTNAME, t.conv 
INTO    #bal_fact 
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK) 
WHERE   t.dt_rep         = @Anchor 
  AND   t.section_name   = N'Срочные' 
  AND   t.block_name     = N'Привлечение ФЛ' 
  AND   t.od_flag        = 1 
  AND   t.cur            = '810' 
  AND   t.out_rub IS NOT NULL; 

/*---------------------------------------------------------------------- 
  2. результирующая таблица (агрегат) 
----------------------------------------------------------------------*/ 
IF OBJECT_ID('WORK.Forecast_BalanceDaily_cases','U') IS NULL 
    CREATE TABLE WORK.Forecast_BalanceDaily_cases 
    ( case_cd varchar(5), dt_rep date, 
      out_rub_total decimal(20,2), rate_con decimal(9,4), 
      CONSTRAINT PK_FBD_cases PRIMARY KEY(case_cd,dt_rep)); 
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDaily_cases; 

/*********************************************************************** 
   если нужен детальный уровень (deals) — снимите комментарии в блоке 2.7 
***********************************************************************/ 

/*====================================================================== 
  2. ЦИКЛ ПО ВСЕМ КЕЙСАМ  (3B, 1A, 2A) 
======================================================================*/ 
DECLARE @case_cd varchar(5), @scen tinyint, @set_cd char(1); 

DECLARE c CURSOR LOCAL FAST_FORWARD FOR 
SELECT case_cd, scen, set_cd FROM #cases ORDER BY case_cd; 

OPEN c; 
FETCH c INTO @case_cd,@scen,@set_cd; 
WHILE @@FETCH_STATUS = 0 
BEGIN 
/*------------------------------------------------------------------ 
  2.1 spot-KEY (TERM=1) для выбранного сценария 
------------------------------------------------------------------*/ 
    IF OBJECT_ID('tempdb..#key_spot') IS NOT NULL DROP TABLE #key_spot; 
    SELECT fc.DT_REP, fc.KEY_RATE 
    INTO   #key_spot 
    FROM   WORK.ForecastKey_Cache_Scen fc 
    JOIN   #cal c ON c.d = fc.DT_REP 
    WHERE  fc.Scenario = @scen 
      AND  fc.TERM     = 1; 

/*------------------------------------------------------------------ 
  2.2 средние KEY-rates на даты открытия (нужны для FIX) 
------------------------------------------------------------------*/ 
    IF OBJECT_ID('tempdb..#key_open') IS NOT NULL DROP TABLE #key_open; 
    SELECT  DT_REP , TERM , AVG_KEY_RATE 
    INTO    #key_open 
    FROM    WORK.ForecastKey_Cache_Scen 
    WHERE   Scenario = @scen 
      AND   DT_REP  BETWEEN '2000-01-01' AND @HorizonTo; 

/*------------------------------------------------------------------ 
  *** 2.3 ref-spread для FIX  (ИСПРАВЛЕННЫЙ БЛОК) *** 
------------------------------------------------------------------*/ 
    /* диапазоны сроков (то, что раньше было захардкожено) */ 
    IF OBJECT_ID('tempdb..#term_rng') IS NOT NULL DROP TABLE #term_rng; 
    CREATE TABLE #term_rng(term_nom int PRIMARY KEY, term_lo int, term_hi int); 
    INSERT #term_rng VALUES 
    ( 61,  46,  76),( 91,  76, 106),(122, 107, 137), 
    (181, 166, 196),(274, 259, 289),(365, 352, 382), 
    (548, 535, 565),(730, 715, 745),(1100,1085,1115); 

    IF OBJECT_ID('tempdb..#ref') IS NOT NULL DROP TABLE #ref; 

    SELECT  s.seg,                         -- 'R' | 'O' 
            t.term_nom, t.term_lo, t.term_hi, 
            tobe_end = o.o_rate,           -- ставка O-клиента 
            key_avg  = k.AVG_KEY_RATE,     -- средний KEY на дату open 
            delta_ro = d.delta_ro          -- Δ(O-R) 
    INTO   #ref 
    FROM  (VALUES ('R'),('O')) s(seg)                 -- два сегмента 
    CROSS JOIN #to_be_O        o 
    JOIN       #term_rng       t ON t.term_nom = o.term_nom 
    JOIN       #delta_R        d ON d.term_nom = o.term_nom 
    JOIN       #key_open       k ON k.DT_REP = @Anchor 
                                 AND k.TERM   = t.term_nom 
    WHERE      o.set_cd = @set_cd;          -- A или B 

    /* добавляем итоговые спреды */ 
    ALTER TABLE #ref ADD 
        spread_end      decimal(9,6), 
        spread_monthly  decimal(9,6); 

    UPDATE #ref 
       SET spread_end = CASE WHEN seg='O' 
                             THEN tobe_end - key_avg 
                             ELSE tobe_end - delta_ro - key_avg END, 
           spread_monthly = CASE 
               WHEN seg='O' AND term_nom BETWEEN 46 AND 196 
                    THEN CAST([LIQUIDITY].[liq].[fnc_IntRate] 
                             (tobe_end,'at the end','monthly',term_nom,1) 
                          AS decimal(9,6)) - key_avg 
               WHEN seg='R' AND term_nom BETWEEN 46 AND 196 
                    THEN CAST([LIQUIDITY].[liq].[fnc_IntRate] 
                             (tobe_end - delta_ro,'at the end','monthly',term_nom,1) 
                          AS decimal(9,6)) - key_avg 
               ELSE NULL END; 

/*------------------------------------------------------------------ 
  2.4 база с spread_final (FIX) и spread_float (FLOAT) 
------------------------------------------------------------------*/ 
    IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base; 

    ;WITH m AS ( 
        SELECT  b.con_id, b.out_rub, b.rate_con, b.is_floatrate, 
                b.termdays, b.dt_open, b.dt_close, b.TSEGMENTNAME, 
                seg         = CASE WHEN b.TSEGMENTNAME=N'Розничный Бизнес' 
                                   THEN 'R' ELSE 'O' END, 
                conv        = b.conv, 
                spread_float = b.rate_con - ks.KEY_RATE, 
                key_avg_open = k.AVG_KEY_RATE 
        FROM    #bal_fact  b 
        LEFT    JOIN #key_spot ks ON ks.DT_REP = @Anchor      -- spot=Anchor 
        LEFT    JOIN #key_open k  ON k.DT_REP = b.dt_open AND k.TERM=b.termdays 
    ) 
    SELECT  m.con_id, m.out_rub, m.is_floatrate, m.termdays, 
            m.dt_open, m.dt_close, m.spread_float, 
            spread_fix_fact = m.rate_con - m.key_avg_open, 
            spread_final = 
                CASE WHEN m.is_floatrate = 1 THEN NULL 
                     ELSE COALESCE( 
                              CASE WHEN m.conv='AT_THE_END' 
                                       THEN r.spread_end 
                                       ELSE ISNULL(r.spread_monthly,r.spread_end) 
                              END, 
                              m.rate_con - m.key_avg_open) 
                END 
    INTO    #base 
    FROM    m 
    LEFT    JOIN #ref r 
           ON r.seg = m.seg 
          AND m.termdays BETWEEN r.term_lo AND r.term_hi; 

/*------------------------------------------------------------------ 
  2.5 roll-over-цепочки (n = 0,1,2,…) 
------------------------------------------------------------------*/ 
    IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls; 
    ;WITH seq AS ( 
        SELECT con_id,out_rub,is_floatrate,termdays,dt_open, 
               spread_float, spread_fix = spread_fix_fact, spread_final, n = 0 
        FROM   #base 
        UNION ALL 
        SELECT con_id,out_rub,is_floatrate,termdays, 
               DATEADD(day,termdays,dt_open), 
               spread_float, spread_final, spread_final, n+1 
        FROM   seq 
        WHERE  DATEADD(day,termdays,dt_open) <= @HorizonTo 
    ) 
    SELECT con_id,out_rub,is_floatrate,termdays, 
           dt_open, dt_close = DATEADD(day,termdays,dt_open), 
           spread_float, spread_fix 
    INTO   #rolls 
    FROM   seq OPTION (MAXRECURSION 0); 

/*------------------------------------------------------------------ 
  2.6 ставки по дням 
------------------------------------------------------------------*/ 
    IF OBJECT_ID('tempdb..#daily') IS NOT NULL DROP TABLE #daily; 
    SELECT c.d AS dt_rep, r.con_id, r.out_rub, 
           rate_con = CASE 
                         WHEN r.is_floatrate = 1 
                              THEN ks.KEY_RATE + r.spread_float 
                         ELSE ISNULL(fk.AVG_KEY_RATE + r.spread_fix, 
                                     r.spread_fix) 
                      END 
    INTO   #daily 
    FROM   #cal   c 
    JOIN   #rolls r ON c.d BETWEEN r.dt_open AND DATEADD(day,-1,r.dt_close) 
    LEFT  JOIN #key_spot ks ON ks.DT_REP = c.d 
    LEFT  JOIN #key_open fk ON fk.DT_REP = r.dt_open AND fk.TERM = r.termdays; 

/*------------------------------------------------------------------ 
  2.7 агрегат и (опц.) деталь 
------------------------------------------------------------------*/ 
    INSERT INTO WORK.Forecast_BalanceDaily_cases(case_cd,dt_rep, 
                                                 out_rub_total,rate_con) 
    SELECT  @case_cd, dt_rep, SUM(out_rub), 
            SUM(out_rub * rate_con) / SUM(out_rub) 
    FROM    #daily 
    GROUP BY dt_rep; 

    /* --- детальный уровень (снимите комментарии, если нужен) 
    IF OBJECT_ID('WORK.Forecast_BalanceDeals_cases','U') IS NULL 
        CREATE TABLE WORK.Forecast_BalanceDeals_cases 
        ( case_cd varchar(5), dt_rep date, con_id bigint, 
          out_rub decimal(20,2), rate_con decimal(9,4), 
          CONSTRAINT PK_FBDc PRIMARY KEY(case_cd,dt_rep,con_id)); 

    INSERT INTO WORK.Forecast_BalanceDeals_cases 
           (case_cd,dt_rep,con_id,out_rub,rate_con) 
    SELECT  @case_cd, dt_rep, con_id, out_rub, rate_con 
    FROM    #daily; 
    ---*/ 

    FETCH c INTO @case_cd,@scen,@set_cd; 
END 
CLOSE c; 
DEALLOCATE c; 

/*---------------------------------------------------------------------- 
  ИТОГО 
----------------------------------------------------------------------*/ 
SELECT * 
FROM   WORK.Forecast_BalanceDaily_cases 
ORDER  BY case_cd, dt_rep; 
