Ниже один T-SQL скрипт, в котором надо только:
	•	вручную задать значения ставок О-сегмента («to-be O») для двух наборов A и B – в блоке /*-- INPUT: TO-BE-O RATES … */;
	•	проверить коэффициент перевода “at-the-end → monthly” – я оставил вызов вашей liq.fnc_IntRate; если имя / сигнатура другие – подправьте.

Скрипт:
	•	берёт уже построенный кэш ключевой ставки по сценарию
(WORK.ForecastKey_Cache_Scen, делаем WHERE SCENARIO = …);
	•	строит справочник spread-ов для FIX-депозитов на основе
– выбранного набора to-be-O,
– разницы ΔRO (О − R) для каждого срока,
– среднего прогнозного key-rate на дату открытия вклада;
	•	для FLOAT-депозитов оставляет факт-спред;
	•	рулит roll-over до горизонта, считает посуточные ставки,
даёт агрегат out_rub_total / rate_con на каждый dt_rep;
	•	один запуск может посчитать несколько случаев сразу
(в примере: 3B, 1A, 2A) – результирующие строки различаются полем case_cd.

⸻


/*----------------------------------------------------------------------
  ПАРАМЕТРЫ ВЫЧИСЛЕНИЯ
----------------------------------------------------------------------*/
DECLARE @Anchor      date = '2025-07-16',      -- последний факт
        @HorizonTo   date = '2025-09-30';      -- конец прогноза

/*----------------------------------------------------------------------
  INPUT:  TO-BE-O RATES  (два набора ― A и B)  + Δ(O-R) по каждому сроку
----------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#to_be_O'      ) IS NOT NULL DROP TABLE #to_be_O;
IF OBJECT_ID('tempdb..#delta_R'      ) IS NOT NULL DROP TABLE #delta_R;
IF OBJECT_ID('tempdb..#cases'        ) IS NOT NULL DROP TABLE #cases;

/*  один набор – одна строка  (O-ставка для срока term_nom) */
CREATE TABLE #to_be_O
( set_cd  char(1),   -- A | B
  term_nom int,      -- 61,91,122,…
  o_rate  decimal(9,4) );

INSERT #to_be_O VALUES
-- set A  (-3 пп в июле, -2 пп в сент.)
('A',61,17.40),('A',91,17.00),('A',122,16.70),('A',181,16.10),
('A',274,15.60),('A',365,15.40),('A',548,14.30),('A',730,14.10),('A',1100,13.60),
-- set B  (-2 пп в июле, -2 пп в сент.)
('B',61,18.40),('B',91,18.00),('B',122,17.50),('B',181,16.60),
('B',274,15.90),('B',365,15.70),('B',548,14.60),('B',730,14.40),('B',1100,13.90);

CREATE TABLE #delta_R   -- Δ(O-R) одинаковая для A и B
(term_nom int PRIMARY KEY,
 delta_ro decimal(9,4));

INSERT #delta_R VALUES
(61,0.50),(91,0.40),(122,0.60),(181,0.40),
(274,0.40),(365,0.20),(548,0.70),(730,0.70),(1100,0.70);

/*  какие «кейсы» нужно посчитать ― сценарий + набор O-ставок */
CREATE TABLE #cases
(case_cd varchar(5) PRIMARY KEY,   -- например 1A, 2A, 3B
 scen tinyint,                     -- номер сценария (1/2/3)
 set_cd char(1));                  -- A | B

INSERT #cases VALUES
('3B',3,'B'),
('1A',1,'A'),
('2A',2,'A');

/*----------------------------------------------------------------------
  0. предварительные календари / key-spot
----------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal)<@HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/*----------------------------------------------------------------------
  1.  факт-портфель на @Anchor  (фикс и флоут вместе)
----------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#bal_fact') IS NOT NULL DROP TABLE #bal_fact;

SELECT  t.con_id,
        t.out_rub,
        t.rate_con,
        t.is_floatrate,
        t.termdays,
        t.dt_open,
        t.dt_close,
        t.TSEGMENTNAME,
        t.conv
INTO    #bal_fact
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE   t.dt_rep=@Anchor
  AND   t.section_name=N'Срочные'
  AND   t.block_name=N'Привлечение ФЛ'
  AND   t.od_flag=1
  AND   t.cur='810'
  AND   t.out_rub IS NOT NULL;

/*----------------------------------------------------------------------
  2. цикл по ВСЕМ случаям (3B,1A,2A)  – строим прогноз и агрегат
----------------------------------------------------------------------*/
IF OBJECT_ID('WORK.Forecast_BalanceDaily_cases','U') IS NULL
CREATE TABLE WORK.Forecast_BalanceDaily_cases
( case_cd varchar(5),
  dt_rep  date,
  out_rub_total decimal(20,2),
  rate_con      decimal(9,4),
  CONSTRAINT PK_FBD_cases PRIMARY KEY(case_cd,dt_rep));
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDaily_cases;

/***********************************************************************
   !!!  если нужен также детальный уровень (deals) –
       раскомментируйте блок в самом конце
***********************************************************************/

DECLARE @case_cd varchar(5), @scen tinyint, @set_cd char(1);

DECLARE c CURSOR LOCAL FAST_FORWARD FOR
SELECT case_cd,scen,set_cd FROM #cases ORDER BY case_cd;

OPEN c;
FETCH c INTO @case_cd,@scen,@set_cd;
WHILE @@FETCH_STATUS=0
BEGIN
    /*--- 2.1 key-спот по выбранному сценарию ----------------------*/
    IF OBJECT_ID('tempdb..#key_spot') IS NOT NULL DROP TABLE #key_spot;
    SELECT fc.DT_REP,fc.KEY_RATE
    INTO   #key_spot
    FROM   WORK.ForecastKey_Cache_Scen fc
    JOIN   #cal c                 ON c.d=fc.DT_REP
    WHERE  fc.Scenario=@scen
      AND  fc.TERM=1;

    /*--- 2.2 средний key-rate на дату открытия  -------------------*/
    IF OBJECT_ID('tempdb..#key_open') IS NOT NULL DROP TABLE #key_open;
    SELECT  DT_REP , TERM , AVG_KEY_RATE
    INTO    #key_open
    FROM    WORK.ForecastKey_Cache_Scen
    WHERE   Scenario=@scen
      AND   DT_REP BETWEEN '2000-01-01' AND @HorizonTo;   -- всё, не только Anchor

    /*--- 2.3 ref-spread для FIX  ----------------------------------*/
    IF OBJECT_ID('tempdb..#ref') IS NOT NULL DROP TABLE #ref;

    SELECT  r.seg,
            r.term_nom,
            r.term_lo,
            r.term_hi,
            tobe_end = o.o_rate,
            key_avg  = k.AVG_KEY_RATE,
            delta_ro = d.delta_ro
    INTO   #ref
    FROM  ( VALUES ('R'),('O') ) r(seg)          -- два сегмента
    CROSS JOIN #to_be_O o
    JOIN      #delta_R d        ON d.term_nom=o.term_nom
    JOIN      #key_open k       ON k.DT_REP=@Anchor  AND k.TERM=o.term_nom
    WHERE     o.set_cd=@set_cd;

    /* добавляем spread_end / spread_monthly */
    ALTER TABLE #ref ADD
        spread_end      decimal(9,6),
        spread_monthly  decimal(9,6);

    UPDATE #ref
       SET spread_end     = CASE WHEN seg='O' THEN tobe_end-key_avg
                                 ELSE tobe_end-delta_ro-key_avg END,
           spread_monthly = CASE
                          WHEN seg='O' AND term_nom BETWEEN 46 AND 196
                              THEN CAST([LIQUIDITY].[liq].[fnc_IntRate]
                                  (tobe_end,'at the end','monthly',term_nom,1)
                                       AS decimal(9,6)) - key_avg
                          WHEN seg='R' AND term_nom BETWEEN 46 AND 196
                              THEN CAST([LIQUIDITY].[liq].[fnc_IntRate]
                                  (tobe_end-delta_ro,'at the end','monthly',term_nom,1)
                                       AS decimal(9,6)) - key_avg
                          ELSE NULL END;

    /*--- 2.4 база с присвоенным spread_final ----------------------*/
    IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;

    ;WITH m AS (
        SELECT  b.con_id,
                b.out_rub,
                b.rate_con,
                b.is_floatrate,
                b.termdays,
                b.dt_open,
                b.dt_close,
                b.TSEGMENTNAME,
                seg = CASE WHEN b.TSEGMENTNAME=N'Розничный Бизнес' THEN 'R' ELSE 'O' END,
                conv = b.conv,
                spread_float = b.rate_con - ks.KEY_RATE,
                key_avg_open = k.AVG_KEY_RATE
        FROM    #bal_fact          b
        LEFT    JOIN #key_spot     ks ON ks.DT_REP=@Anchor           -- spot на Anchor
        LEFT    JOIN #key_open     k  ON k.DT_REP=b.dt_open AND k.TERM=b.termdays
    )
    SELECT  m.con_id,
            m.out_rub,
            m.is_floatrate,
            m.termdays,
            m.dt_open,
            m.dt_close,
            m.spread_float,
            spread_fix_fact = m.rate_con - m.key_avg_open,
            spread_final =
                 CASE WHEN m.is_floatrate=1 THEN NULL
                      ELSE
                           COALESCE(                         -- ищем в ref
                               CASE WHEN m.conv='AT_THE_END'
                                        THEN r.spread_end
                                        ELSE ISNULL(r.spread_monthly,r.spread_end)
                               END,
                               m.rate_con - m.key_avg_open)  -- fallback = факт
                 END
    INTO    #base
    FROM    m
    LEFT    JOIN #ref r
           ON r.seg      = m.seg
          AND m.termdays BETWEEN r.term_lo AND r.term_hi;

    /*--- 2.5 roll-over-цепочки  ----------------------------------*/
    IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;
    ;WITH seq AS (
        SELECT  con_id,out_rub,is_floatrate,termdays,
                dt_open,
                spread_float,
                spread_fix = spread_fix_fact,        -- n=0
                spread_final,
                n = 0
        FROM   #base
        UNION ALL
        SELECT  s.con_id,s.out_rub,s.is_floatrate,s.termdays,
                DATEADD(day,s.termdays,s.dt_open),
                s.spread_float,
                s.spread_final,                      -- n>=1
                s.spread_final,
                n+1
        FROM   seq s
        WHERE  DATEADD(day,s.termdays,s.dt_open) <= @HorizonTo
    )
    SELECT con_id,out_rub,is_floatrate,termdays,
           dt_open,
           dt_close = DATEADD(day,termdays,dt_open),
           spread_float,spread_fix
    INTO   #rolls
    FROM   seq OPTION (MAXRECURSION 0);

    /*--- 2.6 ставки по дням  -------------------------------------*/
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

    /*--- 2.7 агрегат  --------------------------------------------*/
    INSERT INTO WORK.Forecast_BalanceDaily_cases(case_cd,dt_rep,out_rub_total,rate_con)
    SELECT  @case_cd,
            dt_rep,
            SUM(out_rub),
            SUM(out_rub*rate_con)/SUM(out_rub)
    FROM    #daily
    GROUP BY dt_rep;

    /*-- детальный уровень  (раскомментировать при необходимости)
    IF OBJECT_ID('WORK.Forecast_BalanceDeals_cases','U') IS NULL
        CREATE TABLE WORK.Forecast_BalanceDeals_cases
        (case_cd varchar(5),dt_rep date,con_id bigint,
         out_rub decimal(20,2),rate_con decimal(9,4),
         CONSTRAINT PK_FBDc PRIMARY KEY(case_cd,dt_rep,con_id));
    INSERT INTO WORK.Forecast_BalanceDeals_cases
           (case_cd,dt_rep,con_id,out_rub,rate_con)
    SELECT  @case_cd,dt_rep,con_id,out_rub,rate_con
    FROM    #daily;
    --*/

    FETCH c INTO @case_cd,@scen,@set_cd;
END
CLOSE c;
DEALLOCATE c;

/*----------------------------------------------------------------------
  ИТОГО
----------------------------------------------------------------------*/
SELECT *                      -- готовый агрегат
FROM   WORK.Forecast_BalanceDaily_cases
ORDER  BY case_cd, dt_rep;

куда смотреть
	•	WORK.Forecast_BalanceDaily_cases – итог для каждого case_cd
	•	case_cd = '3B'  → сценарий 3 + набор B (-2 пп в июле)
	•	case_cd = '1A'  → сценарий 1 + набор A (-3 пп в июле)
	•	case_cd = '2A'  → сценарий 2 + набор A
	•	если нужен подробный список сделок – раскомментируйте блок с
WORK.Forecast_BalanceDeals_cases.

Скрипт не трогает ваши старые таблицы; создаёт / очищает только временные (#…) и две рабочие (Forecast_BalanceDaily_cases, при желании – Deals_cases).
