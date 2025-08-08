/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2 (FIX-promo-roll, prod_id = 654)
   v.2025-08-09 — без init_rate в рекурсии, без дублей
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO

/* ── параметры ─────────────────────────────────────────────── */
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/* ── 0) Календарь и KEY (TERM=1) ───────────────────────────── */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

DROP TABLE IF EXISTS #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

DECLARE @KeyAnchor decimal(9,4);
SELECT @KeyAnchor = KEY_RATE FROM #key WHERE DT_REP=@Anchor;

/* ── 1) Снимок promo-портфеля (только июль–август-25) ───────── */
DROP TABLE IF EXISTS #bal_prom;
SELECT
    CAST(t.con_id AS bigint)          AS con_id,
    CAST(t.cli_id AS bigint)          AS cli_id,
    CAST(t.out_rub AS decimal(20,2))  AS out_rub,
    CAST(t.rate_con AS decimal(9,4))  AS rate_open,    -- ставка на @Anchor (для cycle 0)
    CAST(t.dt_open AS date)           AS dt_open,
    t.TSEGMENTNAME
INTO #bal_prom
FROM   ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE  t.dt_rep       = @Anchor
  AND  t.prod_id      = 654
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.cur          = '810'
  AND  t.od_flag      = 1
  AND  t.out_rub IS NOT NULL
  AND  t.dt_open BETWEEN '2025-07-01' AND '2025-08-31';

/* ── 2) Постоянные спреды по август-25 открытиям ───────────── */
DROP TABLE IF EXISTS WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(
  TSEGMENTNAME nvarchar(40) PRIMARY KEY,
  spread       decimal(9,4) NOT NULL
);

DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

;WITH aug AS (
    SELECT TSEGMENTNAME,
           w_rate = SUM(out_rub*rate_open)/SUM(out_rub)
    FROM   #bal_prom
    WHERE  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
    GROUP  BY TSEGMENTNAME
)
SELECT
    @Spread_DChbo   = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО'            THEN w_rate END) - @KeyAnchor,
    @Spread_Retail  = MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END) - @KeyAnchor;

INSERT WORK.NS_Spreads VALUES
       (N'ДЧБО', @Spread_DChbo),
       (N'Розничный бизнес', @Spread_Retail);

/* ── 3) Циклы promo: «2 прожитых месяца» на каждую запись ──── */
DROP TABLE IF EXISTS #cycles;
;WITH seed AS (
    SELECT
        p.con_id, p.cli_id, p.TSEGMENTNAME, p.out_rub,
        CAST(0 AS int) AS cycle_no,
        p.dt_open      AS win_start,
        DATEADD(day,-1, EOMONTH(DATEADD(month,1,p.dt_open))) AS win_end
    FROM #bal_prom p
),
seq AS (
    SELECT * FROM seed
    UNION ALL
    SELECT
        s.con_id, s.cli_id, s.TSEGMENTNAME, s.out_rub,
        s.cycle_no + 1,
        DATEADD(day,1, s.win_end) AS win_start,
        DATEADD(day,-1, EOMONTH(DATEADD(month,1, DATEADD(day,1,s.win_end)))) AS win_end
    FROM seq s
    WHERE DATEADD(day,1,s.win_end) <= @HorizonTo
)
SELECT *
INTO   #cycles
FROM   seq
OPTION (MAXRECURSION 0);

/* ── 4) Дневная лента без 1-х чисел ───────────────────────────
      d ∈ [win_start, win_end], внутри окна ставка фиксирована:
        • для cycle 0 → rate_open (#bal_prom),
        • для cycle ≥1 → KEY(win_start)+spread;
      в день win_end → @BaseRate                                     */
DROP TABLE IF EXISTS #daily_pre;

SELECT
    c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub,
    c.cycle_no, c.win_start, c.win_end,
    d.d AS dt_rep,
    CASE
      WHEN d.d < c.win_end THEN
         CASE WHEN c.cycle_no = 0
              THEN bp.rate_open
              ELSE (s.spread + kw.KEY_RATE) END
      WHEN d.d = c.win_end THEN @BaseRate
    END AS rate_con
INTO   #daily_pre
FROM   #cycles c
JOIN   #cal d
       ON d.d BETWEEN c.win_start AND c.win_end
LEFT   JOIN #key kw
       ON kw.DT_REP = c.win_start       -- только для cycle>=1
JOIN   WORK.NS_Spreads s
       ON s.TSEGMENTNAME = c.TSEGMENTNAME
JOIN   #bal_prom bp
       ON bp.con_id = c.con_id;

/* ── 5) Кандидаты на 1-е число (внутренние 1-е и старт новые) ─ */
DROP TABLE IF EXISTS #day1_candidates;

;WITH day1 AS (
    SELECT
        c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub, c.cycle_no,
        c.win_start, c.win_end,
        DATEFROMPARTS(YEAR(d.d),MONTH(d.d),1) AS dt_rep
    FROM   #cycles c
    JOIN   #cal d
      ON   d.d BETWEEN c.win_start AND DATEADD(day,1,c.win_end)
    WHERE  d.d = DATEFROMPARTS(YEAR(d.d),MONTH(d.d),1)
)
SELECT
    y.con_id, y.cli_id, y.TSEGMENTNAME, y.out_rub,
    y.cycle_no, y.win_start, y.win_end,
    y.dt_rep,
    CASE
      /* 1-е число внутри текущего окна */
      WHEN y.dt_rep > y.win_start AND y.dt_rep <= y.win_end
           THEN CASE WHEN y.cycle_no=0
                     THEN bp.rate_open
                     ELSE (s.spread + kw.KEY_RATE) END
      /* 1-е число — старт нового окна (после базового дня) */
      WHEN y.dt_rep = DATEADD(day,1,y.win_end)
           THEN (s.spread + k1.KEY_RATE)
    END AS rate_con
INTO   #day1_candidates
FROM   day1 y
JOIN   #bal_prom bp ON bp.con_id = y.con_id
LEFT   JOIN #key kw ON kw.DT_REP = y.win_start
LEFT   JOIN #key k1 ON k1.DT_REP = y.dt_rep
JOIN   WORK.NS_Spreads s ON s.TSEGMENTNAME = y.TSEGMENTNAME;

/* ── 6) Перелив 1-го числа: весь promo-объём клиента на max ставку ─ */
DROP TABLE IF EXISTS #day1_glue;

;WITH ranked AS (
    SELECT
        c.cli_id, c.dt_rep, c.TSEGMENTNAME, c.out_rub, c.rate_con,
        ROW_NUMBER() OVER (PARTITION BY c.cli_id,c.dt_rep ORDER BY c.rate_con DESC, c.TSEGMENTNAME) AS rn
    FROM #day1_candidates c
)
SELECT
    CAST(NULL AS bigint)              AS con_id,
    r.cli_id,
    CAST(MAX(CASE WHEN rn=1 THEN TSEGMENTNAME END) AS nvarchar(40)) AS TSEGMENTNAME,
    SUM(r.out_rub) AS out_rub,        -- суммарно по клиенту
    r.dt_rep,
    MAX(r.rate_con) AS rate_con
INTO   #day1_glue
FROM   ranked r
GROUP  BY r.cli_id, r.dt_rep;

/* ── 7) Финальная promo-лента и агрегат по дням ─────────────── */
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;

SELECT dt_rep,
       SUM(out_rub)                                 AS out_rub_total,
       SUM(out_rub*rate_con)/NULLIF(SUM(out_rub),0) AS rate_avg
INTO   WORK.Forecast_NS_Promo
FROM (
      /* все дни, кроме 1-х чисел */
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con
      FROM   #daily_pre
      WHERE  dt_rep <> DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)

      UNION ALL

      /* 1-е числа — одна строка на клиента */
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con
      FROM   #day1_glue
) x
GROUP BY dt_rep;

/* ── 8) Promo-ставки (1-е число каждого месяца) ─────────────── */
DROP TABLE IF EXISTS WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates(
  month_first  date,
  TSEGMENTNAME nvarchar(40),
  promo_rate   decimal(9,4),
  PRIMARY KEY(month_first,TSEGMENTNAME)
);

;WITH mkey AS (
    SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) AS m1,
           MIN(KEY_RATE) AS key_min
    FROM   #key
    GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1)
)
INSERT WORK.NS_PromoRates(month_first,TSEGMENTNAME,promo_rate)
SELECT m.m1, s.TSEGMENTNAME, m.key_min + s.spread
FROM   mkey m
JOIN   WORK.NS_Spreads s ON 1=1;

/* ── 9) Контрольный вывод ───────────────────────────────────── */
PRINT N'=== spread (модель, Aug-25) ===';
SELECT * FROM WORK.NS_Spreads;

PRINT N'=== promo-rate (1-е числа) ===';
SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,TSEGMENTNAME;

PRINT N'=== PROMO агрегат (первые 60 дней) ===';
SELECT TOP (60) * FROM WORK.Forecast_NS_Promo ORDER BY dt_rep;
GO
