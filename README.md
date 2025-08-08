/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2  (FIX-promo-roll, prod_id = 654)
   v.2025-08-08 — stand-alone, без зависимостей от Ч.1
   ЛОГИКА:
     • два прожитых календарных месяца promo
     • базовый день = последний день 2-го месяца окна (см. формулу)
     • 1-е число: новая promo-ставка (KEY(1-е)) + перелив внутри cli_id
       ─ победителю даём Σобъём клиента, остальным ставим out_rub=0
   Сохраняет:
     WORK.NS_Spreads, WORK.NS_PromoRates, WORK.Forecast_NS_Promo
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
/* ── параметры ─────────────────────────────────────────────── */
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/* ── календарь горизонта ───────────────────────────────────── */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* ── ключевая ставка spot (TERM=1) ─────────────────────────── */
DROP TABLE IF EXISTS #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

DECLARE @KeyAnchor decimal(9,4);
SELECT @KeyAnchor = KEY_RATE FROM #key WHERE DT_REP=@Anchor;

/* ── портфель: только FIX-счета июль–август-2025 ───────────── */
DROP TABLE IF EXISTS #bal_prom;
SELECT  t.con_id, t.cli_id, t.out_rub, t.rate_con,
        CAST(t.dt_open AS date) AS dt_open,
        t.TSEGMENTNAME
INTO    #bal_prom
FROM    ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE   t.dt_rep       = @Anchor
  AND   t.prod_id      = 654
  AND   t.section_name = N'Накопительный счёт'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.cur          = '810'
  AND   t.od_flag      = 1
  AND   t.out_rub IS NOT NULL
  AND   t.dt_open BETWEEN '2025-07-01' AND '2025-08-31';

/* ── 1) спреды (по открытиям августа-25) ───────────────────── */
DROP TABLE IF EXISTS WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(
  TSEGMENTNAME nvarchar(40) PRIMARY KEY,
  spread       decimal(9,4) NOT NULL);

DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

;WITH aug AS (
    SELECT TSEGMENTNAME,
           w_rate = SUM(out_rub*rate_con)/SUM(out_rub)
    FROM   #bal_prom
    WHERE  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
    GROUP  BY TSEGMENTNAME)
SELECT @Spread_DChbo  = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО'            THEN w_rate END) - @KeyAnchor,
       @Spread_Retail= MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END) - @KeyAnchor
FROM   aug;

INSERT WORK.NS_Spreads VALUES
       (N'ДЧБО',            @Spread_DChbo),
       (N'Розничный бизнес',@Spread_Retail);

/* ── 2) сетка «2 прожитых месяца» для каждого счёта ────────── */
/* Формула win_end = предпоследний день второго месяца.
   Пример: 29-Aug → win_end = 29-Sep.  (Хочешь последний день месяца — скажи.) */
DROP TABLE IF EXISTS #promo_win;
;WITH seed AS (
      SELECT con_id, cli_id, TSEGMENTNAME, out_rub,
             win_start = dt_open,
             win_end   = DATEADD(day,-1, EOMONTH(DATEADD(month,1,dt_open))),
             cyc       = 0
      FROM   #bal_prom)
, seq AS (
      SELECT * FROM seed
      UNION ALL
      SELECT  s.con_id, s.cli_id, s.TSEGMENTNAME, s.out_rub,
              DATEADD(day,1,s.win_end),
              DATEADD(day,-1, EOMONTH(DATEADD(month,1,DATEADD(day,1,s.win_end)))),
              s.cyc+1
      FROM   seq s
      WHERE  DATEADD(day,1,s.win_end) <= @HorizonTo)
SELECT * INTO #promo_win
FROM   seq
OPTION (MAXRECURSION 0);

/* ── 3) raw-дневник: promo-внутри, базовый день, 1-е числа ─── */
DROP TABLE IF EXISTS #FIX_promo_daily_raw;

;WITH grid AS (
    SELECT  p.con_id, p.cli_id, p.TSEGMENTNAME, p.out_rub,
            c.d AS dt_rep, p.win_start, p.win_end
    FROM    #promo_win p
    JOIN    #cal c ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
)
SELECT  g.con_id, g.cli_id, g.TSEGMENTNAME, g.out_rub,
        g.dt_rep,
        rate_con = CASE
            WHEN g.dt_rep <  g.win_end THEN       -- внутри promo-окна
                 CASE g.TSEGMENTNAME
                      WHEN N'ДЧБО'            THEN @Spread_DChbo  + k_open.KEY_RATE
                      ELSE                          @Spread_Retail+ k_open.KEY_RATE END
            WHEN g.dt_rep = g.win_end THEN @BaseRate   -- базовый день
            ELSE                                        -- g.dt_rep = win_end+1 (1-е число)
                 CASE g.TSEGMENTNAME
                      WHEN N'ДЧБО'            THEN @Spread_DChbo  + k_1st.KEY_RATE
                      ELSE                          @Spread_Retail+ k_1st.KEY_RATE END
        END
INTO    #FIX_promo_daily_raw
FROM    grid g
JOIN    #key k_open ON k_open.DT_REP = g.win_start     -- KEY для окна (фикс)
LEFT    JOIN #key k_1st  ON k_1st.DT_REP  = g.dt_rep    -- KEY на 1-е число;

/* ── 4) переразмещение 1-го числа без потери «проигравших» ─── */
/* Выделяем ровно 1-е числа и раскидываем объём: победителю — Σ, проигравшим — 0 */
DROP TABLE IF EXISTS #fix_promo_firstday;
SELECT  r.*
INTO    #fix_promo_firstday
FROM    #FIX_promo_daily_raw r
WHERE   r.dt_rep = DATEFROMPARTS(YEAR(r.dt_rep),MONTH(r.dt_rep),1);

DROP TABLE IF EXISTS #firstday_assigned;
WITH ranked AS (
    SELECT  f.*,
            SUM(f.out_rub) OVER (PARTITION BY f.cli_id)                 AS sum_cli_out,
            ROW_NUMBER() OVER (PARTITION BY f.cli_id ORDER BY f.rate_con DESC, f.con_id) AS rn
    FROM    #fix_promo_firstday f
)
SELECT  con_id, cli_id, TSEGMENTNAME,
        CASE WHEN rn=1 THEN sum_cli_out ELSE CAST(0.0 AS decimal(20,2)) END AS out_rub,
        dt_rep,
        rate_con
INTO    #firstday_assigned
FROM    ranked;

/* Все остальные дни (НЕ 1-е число) остаются как есть */
DROP TABLE IF EXISTS #promo_not_firstday;
SELECT *
INTO   #promo_not_firstday
FROM   #FIX_promo_daily_raw r
WHERE  r.dt_rep <> DATEFROMPARTS(YEAR(r.dt_rep),MONTH(r.dt_rep),1);

/* Финальная promo-лента: объединяем «не 1-е» + «назначенные 1-е» */
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
INTO    WORK.Forecast_NS_Promo
FROM (
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con FROM #promo_not_firstday
      UNION ALL
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con FROM #firstday_assigned
) z
GROUP BY dt_rep;

/* ── 5) promo-ставки «на месяцы» (контроль) ────────────────── */
DROP TABLE IF EXISTS WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates(
  month_first  date,
  TSEGMENTNAME nvarchar(40),
  promo_rate   decimal(9,4),
  PRIMARY KEY(month_first,TSEGMENTNAME));

;WITH mkey AS (
      SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) AS m1,
             MIN(KEY_RATE) AS key_min
      FROM   #key
      GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
INSERT WORK.NS_PromoRates
SELECT m.m1, s.TSEGMENTNAME,
       m.key_min + s.spread
FROM   mkey m
JOIN   WORK.NS_Spreads s ON 1=1;

/* ── 6) вывод ───────────────────────────────────────────────── */
PRINT N'=== spread (Aug-25) ===';
SELECT * FROM WORK.NS_Spreads;

PRINT N'=== promo-rate (1-е) ===';
SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,TSEGMENTNAME;

PRINT N'=== promo-лента (TOP-40) ===';
SELECT TOP (40) * FROM WORK.Forecast_NS_Promo ORDER BY dt_rep, cli_id, con_id;
GO
