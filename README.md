Ниже — полный «Part 2» скрипт с учётом всех замечаний:
	•	промо-окно = ровно два прожитых календарных месяца (29 авг → 29 сент)
	•	base-day = последний день 2-го месяца (@BaseRate)
	•	1-е число нового окна → ставка =KEY(1-е) + спред & glue-перелив на максимальную
	•	во-время promo спот-KEY не переоценивается (ставка фиксируется ключом на dt_open)
	•	из raw-ленты 1-е число удаляется до glue ⇢ дубли объёма исчезают
(объём в итоговой ленте плоский: 30/31 → 1 → 2…)

/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2  (FIX-promo-roll, prod_id = 654)
   v.2025-08-08   —   stand-alone
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
/* ── параметры ─────────────────────────────────────────────── */
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/* ── 0. календарь + KEY-spot ───────────────────────────────── */
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

/* ── 1. портфель promo (июль–август-25) ────────────────────── */
DROP TABLE IF EXISTS #bal_prom;
SELECT  t.con_id,t.cli_id,t.out_rub,t.rate_con,
        CAST(t.dt_open AS date) dt_open,
        t.TSEGMENTNAME
INTO    #bal_prom
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE   t.dt_rep=@Anchor
  AND   t.prod_id      = 654
  AND   t.section_name = N'Накопительный счёт'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.cur          = '810'  AND t.od_flag=1
  AND   t.out_rub IS NOT NULL
  AND   t.dt_open BETWEEN '2025-07-01' AND '2025-08-31';

/* ── 2. спреды по август-25 открытиям ───────────────────────── */
DROP TABLE IF EXISTS WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(
  TSEGMENTNAME nvarchar(40) PRIMARY KEY,
  spread       decimal(9,4));

DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

;WITH aug AS (
        SELECT TSEGMENTNAME,
               w_rate = SUM(out_rub*rate_con)/SUM(out_rub)
        FROM   #bal_prom
        WHERE  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
        GROUP  BY TSEGMENTNAME)
SELECT @Spread_DChbo  = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО'            THEN w_rate END)
                               - (SELECT KEY_RATE FROM #key WHERE DT_REP=@Anchor),
       @Spread_Retail= MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END)
                               - (SELECT KEY_RATE FROM #key WHERE DT_REP=@Anchor)
FROM   aug;

INSERT WORK.NS_Spreads VALUES
       (N'ДЧБО',            @Spread_DChbo),
       (N'Розничный бизнес',@Spread_Retail);

/* ── 3. сетка окон «2 месяца» ───────────────────────────────── */
DROP TABLE IF EXISTS #promo_win;
;WITH seed AS (
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,
             win_start = dt_open,
             win_end   = DATEADD(day,-1,EOMONTH(DATEADD(month,1,dt_open))),
             cyc = 0
      FROM   #bal_prom)
, seq AS (
      SELECT * FROM seed
      UNION ALL
      SELECT  s.con_id,s.cli_id,s.TSEGMENTNAME,s.out_rub,
              DATEADD(day,1,s.win_end),
              DATEADD(day,-1,EOMONTH(DATEADD(month,1,DATEADD(day,1,s.win_end)))),
              s.cyc+1
      FROM   seq s
      WHERE  DATEADD(day,1,s.win_end)<=@HorizonTo)
SELECT * INTO #promo_win FROM seq OPTION (MAXRECURSION 0);

/* ── 4. raw-лента: promo, base-day, 1-е число ──────────────── */
DROP TABLE IF EXISTS #FIX_promo_raw;

SELECT  p.con_id,p.cli_id,p.TSEGMENTNAME,p.out_rub,
        c.d AS dt_rep,
        /* KEY при открытии окна (фиксируем спред) */
        k_open.KEY_RATE              AS key_open,
        /* KEY именно 1-го числа (для нового окна) */
        k_cur.KEY_RATE               AS key_first,
        p.win_start,p.win_end
INTO    #tmp
FROM   #promo_win p
JOIN   #cal  c      ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN   #key k_open  ON k_open.DT_REP = p.win_start
LEFT   JOIN #key k_cur   ON k_cur.DT_REP = c.d
;

/* расчёт ставки + убираем 1-е число ДО glue */
SELECT  con_id,cli_id,TSEGMENTNAME,out_rub,
        dt_rep,
        rate_con =
            CASE
              WHEN dt_rep < win_end
                   THEN CASE TSEGMENTNAME
                          WHEN N'ДЧБО'            THEN @Spread_DChbo  + key_open
                          ELSE                          @Spread_Retail+ key_open END
              WHEN dt_rep = win_end                 -- базовый день
                   THEN @BaseRate
              ELSE                                   /* dt_rep = win_end+1 */
                   CASE TSEGMENTNAME
                      WHEN N'ДЧБО'            THEN @Spread_DChbo  + key_first
                      ELSE                          @Spread_Retail+ key_first END
            END
INTO    #FIX_promo_raw
FROM    #tmp
WHERE   dt_rep <> DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1);  -- 1-е позже

/* ── 5. glue 1-го числа (перелив) ───────────────────────────── */
DROP TABLE IF EXISTS #FIX_promo_glue;
;WITH p1 AS (
      SELECT DISTINCT cli_id,
             m1 = DATEFROMPARTS(YEAR(win_end),MONTH(win_end),1)
      FROM   #promo_win)                   -- строго все 1-е числа
SELECT  CAST(NULL AS bigint)       AS con_id,
        d.cli_id,
        CAST(NULL AS nvarchar(40)) AS TSEGMENTNAME,
        SUM(out_rub)               AS out_rub,
        p.m1                       AS dt_rep,
        MAX(rate_con)              AS rate_con
INTO    #FIX_promo_glue
FROM   #FIX_promo_raw d
JOIN   p1 p ON p.cli_id=d.cli_id AND p.m1=d.dt_rep
GROUP  BY d.cli_id,p.m1;

/* ── 6. финальная promo-лента и агрегат ─────────────────────── */
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;
SELECT dt_rep,
       SUM(out_rub)                       out_rub_total,
       SUM(out_rub*rate_con)/SUM(out_rub) rate_avg
INTO   WORK.Forecast_NS_Promo
FROM (
      SELECT * FROM #FIX_promo_raw
      UNION ALL
      SELECT * FROM #FIX_promo_glue
) x
GROUP BY dt_rep;

/* ── 7. promo-ставка 1-го числа (контроль) ─────────────────── */
DROP TABLE IF EXISTS WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates(
  month_first date,
  TSEGMENTNAME nvarchar(40),
  promo_rate   decimal(9,4),
  PRIMARY KEY(month_first,TSEGMENTNAME));

;WITH mkey AS (
      SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) m1,
             MIN(KEY_RATE) key_min
      FROM   #key
      GROUP BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
INSERT WORK.NS_PromoRates
SELECT m.m1,
       s.TSEGMENTNAME,
       m.key_min + s.spread
FROM   mkey m
JOIN   WORK.NS_Spreads s ON 1=1;

/* ── 8. вывод ───────────────────────────────────────────────── */
PRINT N'=== spread (Aug-25) ===';         SELECT * FROM WORK.NS_Spreads;
PRINT N'=== promo-rate (1-е) ===';        SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,TSEGMENTNAME;
PRINT N'=== promo-лента (TOP-40) ===';
SELECT TOP (40) * FROM WORK.Forecast_NS_Promo ORDER BY dt_rep;
GO

Изменено по сравнению с предыдущей версией
	1.	Ставка внутри окна
@Spread + key_open – ключ на день открытия окна → спред не «прыгает» 15 сентября.
	2.	Удалён дубль 1-го числа из raw ещё до glue (WHERE dt_rep <> 1-е).
Теперь объём не «взлетает» 1-го и не «падает» 2-го.
	3.	Ключ на 1-е число берётся через key_first (LEFT JOIN #key).

Проверьте:

SELECT dt_rep,out_rub_total,rate_avg
FROM   WORK.Forecast_NS_Promo
WHERE  dt_rep BETWEEN '2025-08-25' AND '2025-09-05';

должно дать:

| 30 Aug | … | 6.50 % |
| 31 Aug | … | 6.50 % |
| 01 Sep | … | 17.х % (объём тот же) |
| 02 Sep | … | 17.х % |

и аналогично на стыке сентября/октября и далее.
