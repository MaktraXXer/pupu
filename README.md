


помнишь мой код для расчета спредов?
где был справочник
вот код
### Где ломалось  и что надо поправить

1. **В `#base`** мы переименовали поле со спредом факта в `spread_fix_fact`,
   но дальше, в `#base2`, по-прежнему обращались к старому имени `spread_fix`.
   ⇒ «Недопустимое имя столбца "spread\_fix"».

2. После этого столбца `spread_final` в `#base2` тоже не появлялось,
   поэтому в рекурсивной CTE `seq` возникла вторая ошибка.

Ниже — **цельный рабочий скрипт** v-3 FIN с обеими правками.
(Всё остальное оставлено без изменений.)

```sql
/**************************************************************************
   V-3  FIN  (19 июл 2025)
   • TO-BE-справочник (±15 дн)
   • 31±10 дн → 91 дн
   • fix-депо: spread_final  ←  TO-BE | факт
   • float-депо: только факт-спред
**************************************************************************/
USE ALM_TEST;
GO
DECLARE
    @Anchor     date = '2025-07-16',          -- последний факт
    @HorizonTo  date = '2025-09-30';          -- конец прогноза
/* ---------- 0. справочник TO-BE / KEY ---------- */
IF OBJECT_ID('tempdb..#ref_spread') IS NOT NULL DROP TABLE #ref_spread;
CREATE TABLE #ref_spread
( seg char(1), term_nom int, term_lo int, term_hi int,
  tobe_end decimal(9,6), key_avg decimal(9,6));
INSERT #ref_spread VALUES
('R',  61,  46,  76, 0.1820, 0.174918), ('R',  91,  76, 106, 0.1790, 0.173098),
('R', 122, 107, 137, 0.1810, 0.169306), ('R', 181, 166, 196, 0.1750, 0.163700),
('R', 274, 259, 289, 0.1610, 0.156300), ('R', 367, 352, 382, 0.1610, 0.150200),
('R', 550, 535, 565, 0.1430, 0.142100), ('R', 730, 715, 745, 0.1410, 0.137800),
('R',1100,1085,1115, 0.1360, 0.133200),
('O',  61,  46,  76, 0.1860, 0.174918), ('O',  91,  76, 106, 0.1830, 0.173098),
('O', 122, 107, 137, 0.1850, 0.169306), ('O', 181, 166, 196, 0.1790, 0.163700),
('O', 274, 259, 289, 0.1660, 0.156300), ('O', 367, 352, 382, 0.1660, 0.150200),
('O', 550, 535, 565, 0.1500, 0.142100), ('O', 730, 715, 745, 0.1480, 0.137800),
('O',1100,1085,1115, 0.1430, 0.133200);
/* ---------- 1. календарь и spot-KEY ---------- */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal)<@HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

IF OBJECT_ID('tempdb..#key_spot') IS NOT NULL DROP TABLE #key_spot;
SELECT fc.DT_REP,fc.KEY_RATE INTO #key_spot
FROM WORK.ForecastKey_Cache fc JOIN #cal c ON c.d=fc.DT_REP WHERE fc.TERM=1;
/* ---------- 2. fact-портфель на 16-07 ---------- */
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
SELECT  t.con_id,t.out_rub,t.rate_con,t.is_floatrate,
        t.termdays,t.dt_open,t.dt_close,
        t.TSEGMENTNAME,t.conv,
        spread_float = CASE WHEN t.is_floatrate=1
                                 THEN t.rate_con-ks.KEY_RATE END,
        spread_fix_fact = CASE WHEN t.is_floatrate=0
                                 THEN t.rate_con-fk_open.AVG_KEY_RATE END
INTO    #base
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
JOIN    #key_spot ks ON ks.DT_REP=@Anchor
LEFT    JOIN WORK.ForecastKey_Cache fk_open
           ON fk_open.DT_REP=TRY_CAST(t.DT_OPEN AS date)
          AND fk_open.TERM=t.termdays
WHERE   t.dt_rep=@Anchor AND t.section_name=N'Срочные'
  AND   t.block_name=N'Привлечение ФЛ' AND t.od_flag=1
  AND   t.cur='810' AND t.out_rub IS NOT NULL;
/* ---------- 3. spread_final для FIX ---------- */
IF OBJECT_ID('tempdb..#fix_spread') IS NOT NULL DROP TABLE #fix_spread;
WITH m AS (
    SELECT con_id,termdays,conv,out_rub,
           seg = CASE WHEN TSEGMENTNAME=N'Розничный Бизнес' THEN 'R' ELSE 'O' END,
           term_use = CASE WHEN termdays BETWEEN 21 AND 41 THEN 91 ELSE termdays END
    FROM   #base WHERE is_floatrate=0
),
match_ref AS (
    SELECT m.con_id,r.*,m.conv
    FROM   m JOIN #ref_spread r
      ON  r.seg=m.seg AND m.term_use BETWEEN r.term_lo AND r.term_hi
)
SELECT con_id,
       spread_final = CASE
            WHEN conv='AT_THE_END'
                 THEN tobe_end-key_avg
            ELSE CAST([LIQUIDITY].[liq].[fnc_IntRate]
                      (tobe_end,'at the end','monthly',term_nom,1) AS decimal(9,6))
                 - key_avg
       END
INTO   #fix_spread
FROM   match_ref;
/* ---------- 4. база n=0 + spread_final ---------- */
IF OBJECT_ID('tempdb..#base2') IS NOT NULL DROP TABLE #base2;
SELECT  b.con_id,b.out_rub,b.is_floatrate,b.termdays,b.dt_open,
        b.spread_float,
        b.spread_fix_fact,
        COALESCE(fs.spread_final,b.spread_fix_fact) AS spread_final
INTO    #base2
FROM    #base b
LEFT    JOIN #fix_spread fs ON fs.con_id=b.con_id;
/* ---------- 5. roll-over без OUTER JOIN ---------- */
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;
;WITH seq AS (
    SELECT con_id,out_rub,is_floatrate,termdays,dt_open,
           spread_float,
           spread_fix = spread_fix_fact,   -- n=0
           spread_final,
           n = 0
    FROM   #base2
    UNION ALL
    SELECT con_id,out_rub,is_floatrate,termdays,
           DATEADD(day,termdays,dt_open),
           spread_float,
           spread_final,                    -- n≥1
           spread_final,
           n+1
    FROM   seq
    WHERE  DATEADD(day,termdays,dt_open)<=@HorizonTo
)
SELECT con_id,out_rub,is_floatrate,termdays,
       dt_open,
       dt_close = DATEADD(day,termdays,dt_open),
       spread_float,spread_fix,n
INTO   #rolls
FROM   seq OPTION (MAXRECURSION 0);
/* ---------- 6. посуточные ставки ---------- */
IF OBJECT_ID('tempdb..#daily') IS NOT NULL DROP TABLE #daily;
SELECT c.d AS dt_rep,
       r.con_id,r.out_rub,
       rate_con = CASE
                     WHEN r.is_floatrate=1 THEN ks.KEY_RATE+r.spread_float
                     ELSE ISNULL(fko.AVG_KEY_RATE+r.spread_fix,r.spread_fix)
                  END
INTO   #daily
FROM   #cal c
JOIN   #rolls r ON c.d BETWEEN r.dt_open AND DATEADD(day,-1,r.dt_close)
LEFT  JOIN #key_spot ks ON ks.DT_REP=c.d
LEFT  JOIN WORK.ForecastKey_Cache fko
       ON fko.DT_REP=r.dt_open AND fko.TERM=r.termdays;
/* ---------- 7. выгрузка v3 ---------- */
IF OBJECT_ID('WORK.Forecast_BalanceDaily_v3','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDaily_v3
    (dt_rep DATE PRIMARY KEY,out_rub_total DECIMAL(20,2),rate_con DECIMAL(9,4))
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDaily_v3;

IF OBJECT_ID('WORK.Forecast_BalanceDeals_v3','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDeals_v3
    (dt_rep DATE,con_id BIGINT,out_rub DECIMAL(20,2),rate_con DECIMAL(9,4),
     CONSTRAINT PK_FBD_v3 PRIMARY KEY(dt_rep,con_id))
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDeals_v3;

INSERT INTO WORK.Forecast_BalanceDaily_v3
SELECT dt_rep,SUM(out_rub),SUM(out_rub*rate_con)/SUM(out_rub)
FROM   #daily GROUP BY dt_rep;

вот новый справочник
РОЗНИЦА
Срок	TO BE	Средний прогнозный ключ сейчас
61	18.20%	17.33%
91	17.90%	17.18%
122	18.10%	16.83%
181	17.50%	16.36%
274	16.10%	15.48%
367	16.10%	14.84%
550	14.30%	14.08%
730	14.10%	13.70%
1100	13.60%	13.20%

ДЧБО	
Срок	TO BE	Средний прогнозный ключ сейчас
61	18,60%	17.33%
91	18,30%	17.18%
122	18,50%	16.83%
181	17,90%	16.36%
274	16,60%	15.48%
367	16,60%	14.84%
550	15,00%	14.08%
730	14,80%	13.70%
1100	14,30%	13.20%
