Окей, вот **полный монолит** (Part 2) прямо здесь в чате — с учётом всех твоих правил и с корректной обработкой случаев, когда у клиента 1, 3, 5… кандидатов на 1-е число.

```sql
/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2 (FIX-promo-roll, prod_id = 654, is_floatrate=0)
   v.2025-08-09 — без дублей объёма, базовый день, перелив 1-го, m-кандидатов
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO

/* ── параметры ─────────────────────────────────────────────── */
DECLARE
    @Scenario  tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;   -- 6.5%

/* ── 0) Календарь горизонта + KEY(TERM=1) ───────────────────── */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

DROP TABLE IF EXISTS #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@Scenario AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

DECLARE @KeyAtAnchor decimal(9,4);
SELECT @KeyAtAnchor = KEY_RATE FROM #key WHERE DT_REP = @Anchor;

/* ── 1) Снимок promo-портфеля (FIX, июль–август-2025) ───────── */
DROP TABLE IF EXISTS #bal_prom;
SELECT
    CAST(t.con_id  AS bigint)         AS con_id,
    CAST(t.cli_id  AS bigint)         AS cli_id,
    CAST(t.out_rub AS decimal(20,2))  AS out_rub,
    CAST(t.rate_con AS decimal(9,4))  AS rate_anchor,   -- фактическая ставка на Anchor
    CAST(t.dt_open AS date)           AS dt_open,
    t.TSEGMENTNAME
INTO #bal_prom
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep       = @Anchor
  AND t.prod_id      = 654
  AND t.section_name = N'Накопительный счёт'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.cur          = '810'
  AND t.od_flag      = 1
  AND COALESCE(t.is_floatrate,0) = 0     -- только FIX, без float
  AND t.out_rub IS NOT NULL
  AND t.dt_open BETWEEN '2025-07-01' AND '2025-08-31';

DECLARE @PromoTotal decimal(20,2) = (SELECT SUM(out_rub) FROM #bal_prom);

/* ── 2) Фиксируем спреды по август-25 открытиям ─────────────── */
DROP TABLE IF EXISTS WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(
  TSEGMENTNAME nvarchar(40) PRIMARY KEY,
  spread       decimal(9,4) NOT NULL
);

DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

;WITH aug AS (
    SELECT TSEGMENTNAME,
           w_rate = SUM(out_rub*rate_anchor)/SUM(out_rub)
    FROM   #bal_prom
    WHERE  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
    GROUP  BY TSEGMENTNAME
)
SELECT
    @Spread_DChbo   = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО'            THEN w_rate END) - @KeyAtAnchor,
    @Spread_Retail  = MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END) - @KeyAtAnchor;

INSERT WORK.NS_Spreads VALUES
       (N'ДЧБО',            @Spread_DChbo),
       (N'Розничный бизнес',@Spread_Retail);

/* ── 3) Циклы promo: «2 прожитых месяца» для каждого счёта ─── */
DROP TABLE IF EXISTS #cycles;
;WITH seed AS (
    SELECT
        b.con_id, b.cli_id, b.TSEGMENTNAME, b.out_rub,
        CAST(0 AS int) AS cycle_no,
        b.dt_open AS win_start,
        /* конец promo: последний день следующего месяца от dt_open
           пример: 04-авг → win_end = 29-сен, 30-сен = базовый день */
        DATEADD(day,-1, EOMONTH(DATEADD(month,1,b.dt_open))) AS win_end
    FROM #bal_prom b
),
seq AS (
    SELECT * FROM seed
    UNION ALL
    SELECT
        s.con_id, s.cli_id, s.TSEGMENTNAME, s.out_rub,
        s.cycle_no + 1,
        DATEADD(day,1, s.win_end) AS win_start,  -- 1-е число следующего месяца
        DATEADD(day,-1, EOMONTH(DATEADD(month,1, DATEADD(day,1,s.win_end)))) AS win_end
    FROM seq s
    WHERE DATEADD(day,1,s.win_end) <= @HorizonTo
)
SELECT *
INTO   #cycles
FROM   seq
OPTION (MAXRECURSION 0);

/* ── 4) Дневная лента без 1-х чисел (внутри окна + base-day) ──
       Внутри окна:
         • cycle 0 — держим фактическую ставку на Anchor (rate_anchor)
         • cycle ≥1 — KEY(win_start)+spread (фикс на весь цикл)
       В день win_end — @BaseRate
*/
DROP TABLE IF EXISTS #daily_pre;

SELECT
    c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub,
    c.cycle_no, c.win_start, c.win_end,
    d.d AS dt_rep,
    CASE
      WHEN d.d < c.win_end THEN
           CASE WHEN c.cycle_no = 0
                THEN bp.rate_anchor
                ELSE (CASE c.TSEGMENTNAME
                        WHEN N'ДЧБО'            THEN @Spread_DChbo  + k_open.KEY_RATE
                        ELSE                          @Spread_Retail+ k_open.KEY_RATE
                      END)
           END
      WHEN d.d = c.win_end THEN @BaseRate
    END AS rate_con
INTO   #daily_pre
FROM   #cycles c
JOIN   #cal d
       ON d.d BETWEEN c.win_start AND c.win_end
LEFT   JOIN #key k_open
       ON k_open.DT_REP = c.win_start
JOIN   #bal_prom bp
       ON bp.con_id = c.con_id;

/* ── 5) Кандидаты на 1-е числа и их ставки ────────────────────
       m1_in  — 1-е число ВНУТРИ текущего окна (второй месяц)
       m1_new — 1-е число НОВОГО окна (после base-day)
       На 1-е может быть 1, 3, 5… кандидатов — все попадают в пул.
*/
DROP TABLE IF EXISTS #day1_candidates;

WITH marks AS (
    SELECT
      c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub, c.cycle_no, c.win_start, c.win_end,
      DATEFROMPARTS(YEAR(DATEADD(month,1,c.win_start)),
                    MONTH(DATEADD(month,1,c.win_start)), 1) AS m1_in,
      DATEADD(day,1,c.win_end) AS m1_new
    FROM #cycles c
),
cand AS (
    /* 1-е внутри окна (фикс цикла) */
    SELECT
        m.con_id, m.cli_id, m.TSEGMENTNAME, m.out_rub, m.cycle_no,
        dt_rep = m.m1_in,
        rate_con =
            CASE WHEN m.cycle_no=0
                 THEN bp.rate_anchor
                 ELSE (CASE m.TSEGMENTNAME
                        WHEN N'ДЧБО'            THEN @Spread_DChbo  + k_open.KEY_RATE
                        ELSE                          @Spread_Retail+ k_open.KEY_RATE
                      END)
            END
    FROM marks m
    JOIN #bal_prom bp ON bp.con_id = m.con_id
    LEFT JOIN #key k_open ON k_open.DT_REP = m.win_start
    WHERE m.m1_in BETWEEN m.win_start AND m.win_end
      AND m.m1_in BETWEEN @Anchor AND @HorizonTo

    UNION ALL

    /* 1-е нового окна (новая promo по KEY(1-е)+spread) */
    SELECT
        m.con_id, m.cli_id, m.TSEGMENTNAME, m.out_rub, m.cycle_no + 1 AS cycle_no,
        dt_rep = m.m1_new,
        rate_con =
            CASE m.TSEGMENTNAME
              WHEN N'ДЧБО'            THEN @Spread_DChbo  + k_new.KEY_RATE
              ELSE                          @Spread_Retail+ k_new.KEY_RATE
            END
    FROM marks m
    LEFT JOIN #key k_new ON k_new.DT_REP = m.m1_new
    WHERE m.m1_new BETWEEN @Anchor AND @HorizonTo
)
SELECT * INTO #day1_candidates FROM cand;

/* Суммарный promo-объём клиента на Anchor — инвариант для 1-х чисел */
DROP TABLE IF EXISTS #cli_sum;
SELECT cli_id, SUM(out_rub) AS out_rub_sum
INTO   #cli_sum
FROM   #bal_prom
GROUP  BY cli_id;

/* ── 6) Перелив 1-го числа: один победитель на клиента ────────
       • Ранжируем всех кандидатов по rate_con DESC
       • Победителю отдаём Σобъём клиента (#cli_sum)
       • Остальные в этот день «с нулём» (в агрегат не попадают)
       • При равенстве: детерминированный tie-breaker
*/
DROP TABLE IF EXISTS #firstday_assigned;

WITH ranked AS (
    SELECT
        c.*,
        ROW_NUMBER() OVER (PARTITION BY c.cli_id, c.dt_rep
                           ORDER BY c.rate_con DESC, c.TSEGMENTNAME, c.con_id) AS rn
    FROM #day1_candidates c
)
SELECT
    con_id        = MAX(CASE WHEN rn=1 THEN con_id END),
    r.cli_id,
    TSEGMENTNAME  = MAX(CASE WHEN rn=1 THEN TSEGMENTNAME END),
    out_rub       = cs.out_rub_sum,
    r.dt_rep,
    rate_con      = MAX(CASE WHEN rn=1 THEN rate_con END)
INTO   #firstday_assigned
FROM   ranked r
JOIN   #cli_sum cs ON cs.cli_id = r.cli_id
GROUP  BY r.cli_id, r.dt_rep, cs.out_rub_sum;

/* ── 7) Финальная promo-лента (без дублей объёма) ───────────── */
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;

SELECT
    dt_rep,
    out_rub_total = SUM(out_rub),
    rate_avg      = SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM (
    /* все даты КРОМЕ 1-х чисел — живём на собственных окнах */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #daily_pre
    WHERE  DAY(dt_rep) <> 1

    UNION ALL

    /* 1-е числа — одна строка на клиента, Σобъём, max ставка */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #firstday_assigned
) u
GROUP BY dt_rep;

/* ── 8) Контрольные promo-ставки (1-е каждого месяца) ───────── */
DROP TABLE IF EXISTS WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates(
  month_first  date,
  TSEGMENTNAME nvarchar(40),
  promo_rate   decimal(9,4),
  PRIMARY KEY (month_first, TSEGMENTNAME)
);

;WITH mkey AS (
    SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) AS m1,
           MIN(KEY_RATE) AS key_min
    FROM   #key
    GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1)
)
INSERT WORK.NS_PromoRates(month_first, TSEGMENTNAME, promo_rate)
SELECT m.m1, s.TSEGMENTNAME, m.key_min + s.spread
FROM   mkey m
JOIN   WORK.NS_Spreads s ON 1=1;

/* ── 9) Быстрые проверки ────────────────────────────────────── */
PRINT N'=== spreads (Aug-25) ===';        SELECT * FROM WORK.NS_Spreads;
PRINT N'=== promo-rates (1-е) ===';       SELECT * FROM WORK.NS_PromoRates ORDER BY month_first, TSEGMENTNAME;

PRINT N'=== sanity: тотал должен быть постоянным (= @PromoTotal) ===';
SELECT TOP (60)
       f.dt_rep, f.out_rub_total,
       diff_vs_anchor = CAST(f.out_rub_total - @PromoTotal AS decimal(20,2))
FROM   WORK.Forecast_NS_Promo f
ORDER  BY f.dt_rep;

PRINT N'=== window check: 2025-08-29 .. 2025-10-02 ===';
SELECT dt_rep, out_rub_total, rate_avg
FROM   WORK.Forecast_NS_Promo
WHERE  dt_rep BETWEEN '2025-08-29' AND '2025-10-02'
ORDER  BY dt_rep;
GO
```

Если хочешь поменять политику при равенстве ставок (например, «предпочитать текущего держателя»), скажи — подправлю `ORDER BY` в блоке `ranked`.
