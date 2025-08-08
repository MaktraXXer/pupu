/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2 (FIX-promo-roll, prod_id = 654, is_floatrate=0)
   v.2025-08-10 — base-day half-shift + full shift on 1st; no dupes
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO

/* ── параметры ─────────────────────────────────────────────── */
DECLARE
    @Scenario  tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;  -- 6.5%

/* ── 0) Календарь горизонта + KEY(TERM=1) ───────────────────── */
DROP TABLE IF EXISTS #cal;
SELECT d = @Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

DROP TABLE IF EXISTS #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario = @Scenario AND TERM = 1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

DECLARE @KeyAtAnchor decimal(9,4);
SELECT @KeyAtAnchor = KEY_RATE FROM #key WHERE DT_REP = @Anchor;

/* ── 1) Снимок promo-портфеля (FIX июль–август-2025) ────────── */
DROP TABLE IF EXISTS #bal_prom;
SELECT
    CAST(t.con_id  AS bigint)         AS con_id,
    CAST(t.cli_id  AS bigint)         AS cli_id,
    CAST(t.out_rub AS decimal(20,2))  AS out_rub,
    CAST(t.rate_con AS decimal(9,4))  AS rate_anchor,  -- ставка на Anchor
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
  AND COALESCE(t.is_floatrate,0) = 0
  AND t.out_rub IS NOT NULL
  AND t.dt_open BETWEEN '2025-07-01' AND '2025-08-31';

DECLARE @PromoTotal decimal(20,2) = (SELECT SUM(out_rub) FROM #bal_prom);

/* ── 2) Фикс. спреды по август-25 открытиям ─────────────────── */
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

/* ── 3) Циклы: 2 прожитых месяца; base_day = EOM(next), promo_end = base_day-1 ─ */
DROP TABLE IF EXISTS #cycles;
;WITH seed AS (
    SELECT
        b.con_id, b.cli_id, b.TSEGMENTNAME, b.out_rub,
        CAST(0 AS int) AS cycle_no,
        b.dt_open AS win_start,
        EOMONTH(DATEADD(month,1, b.dt_open))                   AS base_day,   -- последний день
        DATEADD(day,-1, EOMONTH(DATEADD(month,1, b.dt_open)))  AS promo_end   -- предпоследний
    FROM #bal_prom b
),
seq AS (
    SELECT * FROM seed
    UNION ALL
    SELECT
        s.con_id, s.cli_id, s.TSEGMENTNAME, s.out_rub,
        s.cycle_no + 1,
        DATEADD(day,1, s.base_day) AS win_start,               -- 1-е след. месяца
        EOMONTH(DATEADD(month,1, DATEADD(day,1, s.base_day))) AS base_day,
        DATEADD(day,-1, EOMONTH(DATEADD(month,1, DATEADD(day,1, s.base_day)))) AS promo_end
    FROM seq s
    WHERE DATEADD(day,1, s.base_day) <= @HorizonTo
)
SELECT * INTO #cycles FROM seq OPTION (MAXRECURSION 0);

/* ── 4) Дневная лента без 1-х чисел (внутри promo + base-day как есть) ─ */
DROP TABLE IF EXISTS #daily_pre;
SELECT
    c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub,
    c.cycle_no, c.win_start, c.promo_end, c.base_day,
    d.d AS dt_rep,
    CASE
      WHEN d.d <= c.promo_end THEN
           CASE WHEN c.cycle_no = 0
                THEN bp.rate_anchor
                ELSE (CASE c.TSEGMENTNAME
                        WHEN N'ДЧБО'            THEN @Spread_DChbo  + k_open.KEY_RATE
                        ELSE                          @Spread_Retail+ k_open.KEY_RATE
                     END)
           END
      WHEN d.d = c.base_day THEN @BaseRate
    END AS rate_con
INTO   #daily_pre
FROM   #cycles c
JOIN   #cal d
       ON d.d BETWEEN c.win_start AND c.base_day
LEFT   JOIN #key k_open
       ON k_open.DT_REP = c.win_start
JOIN   #bal_prom bp
       ON bp.con_id = c.con_id
WHERE  DAY(d.d) <> 1;   -- 1-е числа делаем отдельно

/* ── 5) BASE-DAY HALF-SHIFT: в последний день месяца переливаем 50% с "базы" на макс ставку ─ */
DROP TABLE IF EXISTS #base_dates;
SELECT DISTINCT dp.cli_id, dp.dt_rep
INTO   #base_dates
FROM   #daily_pre dp
JOIN   #cycles c ON c.con_id=dp.con_id AND c.cycle_no=dp.cycle_no
WHERE  dp.dt_rep = c.base_day;  -- есть хотя бы один счёт с базой у клиента в этот день

DROP TABLE IF EXISTS #base_pool;
SELECT
    dp.*,
    is_base = CASE WHEN dp.dt_rep = c.base_day THEN 1 ELSE 0 END
INTO   #base_pool
FROM   #daily_pre dp
JOIN   #base_dates bd ON bd.cli_id=dp.cli_id AND bd.dt_rep=dp.dt_rep
LEFT  JOIN #cycles c  ON c.con_id=dp.con_id AND c.cycle_no=dp.cycle_no;

DROP TABLE IF EXISTS #daily_base_adj;
WITH sums AS (
    SELECT cli_id, dt_rep,
           shift_amt = CAST(0.5 * SUM(CASE WHEN is_base=1 THEN out_rub ELSE 0 END) AS decimal(20,2))
    FROM   #base_pool
    GROUP  BY cli_id, dt_rep
),
ranked AS (
    SELECT bp.*,
           ROW_NUMBER() OVER (PARTITION BY bp.cli_id, bp.dt_rep
                              ORDER BY bp.rate_con DESC, bp.TSEGMENTNAME, bp.con_id) AS rn
    FROM   #base_pool bp
)
SELECT
    r.con_id,
    r.cli_id,
    r.TSEGMENTNAME,
    CASE
      WHEN r.is_base=1 THEN CAST(r.out_rub * 0.5 AS decimal(20,2))               -- половину оставили на базе
      WHEN r.rn=1     THEN CAST(r.out_rub + s.shift_amt AS decimal(20,2))        -- победителю добавили перелив
      ELSE                  r.out_rub                                           -- остальные без изменений
    END AS out_rub,
    r.dt_rep,
    r.rate_con
INTO   #daily_base_adj
FROM   ranked r
JOIN   sums   s ON s.cli_id=r.cli_id AND s.dt_rep=r.dt_rep;

/* ── 6) Кандидаты на 1-е числа (внутренние и новые окна) ────── */
DROP TABLE IF EXISTS #day1_candidates;

WITH marks AS (
    SELECT
      c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub, c.cycle_no, c.win_start, c.promo_end, c.base_day,
      DATEFROMPARTS(YEAR(DATEADD(month,1,c.win_start)), MONTH(DATEADD(month,1,c.win_start)), 1) AS m1_in,
      DATEADD(day,1,c.base_day) AS m1_new
    FROM #cycles c
),
cand AS (
    /* 1-е внутри текущего окна (если попадает) — ставка фикс цикла */
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
    WHERE m.m1_in BETWEEN m.win_start AND m.promo_end
      AND m.m1_in BETWEEN @Anchor AND @HorizonTo

    UNION ALL

    /* 1-е нового окна — новая promo: KEY(1-е)+spread */
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

/* Σ-объём клиента на Anchor — для 1-х чисел (полный перелив) */
DROP TABLE IF EXISTS #cli_sum;
SELECT cli_id, SUM(out_rub) AS out_rub_sum
INTO   #cli_sum
FROM   #bal_prom
GROUP  BY cli_id;

/* ── 7) Перелив 1-го числа: выбираем max ставку, кладём Σобъём ─ */
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
    out_rub       = cs.out_rub_sum,        -- весь promo-объём клиента
    r.dt_rep,
    rate_con      = MAX(CASE WHEN rn=1 THEN rate_con END)
INTO   #firstday_assigned
FROM   ranked r
JOIN   #cli_sum cs ON cs.cli_id = r.cli_id
GROUP  BY r.cli_id, r.dt_rep, cs.out_rub_sum;

/* ── 8) Финальная promo-лента (без дублей) ──────────────────── */
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;

SELECT
    dt_rep,
    out_rub_total = SUM(out_rub),
    rate_avg      = SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM (
    /* все дни КРОМЕ 1-х и base-day, которые заменяем на adj */
    SELECT dp.con_id, dp.cli_id, dp.TSEGMENTNAME, dp.out_rub, dp.dt_rep, dp.rate_con
    FROM   #daily_pre dp
    WHERE  NOT EXISTS (SELECT 1 FROM #base_dates bd WHERE bd.cli_id=dp.cli_id AND bd.dt_rep=dp.dt_rep)

    UNION ALL
    /* base-day — берём скорректированные (half-shift) */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #daily_base_adj

    UNION ALL
    /* 1-е числа — полный перелив (одна строка на клиента) */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #firstday_assigned
) u
GROUP BY dt_rep;

/* ── 9) Контрольки ──────────────────────────────────────────── */
PRINT N'=== spreads (Aug-25) ===';  SELECT * FROM WORK.NS_Spreads;
PRINT N'=== sanity total (= @PromoTotal) ===';
SELECT TOP (90) f.dt_rep, f.out_rub_total,
       diff_vs_anchor = CAST(f.out_rub_total - @PromoTotal AS decimal(20,2))
FROM WORK.Forecast_NS_Promo f ORDER BY f.dt_rep;

PRINT N'=== window check: 2025-08-29 .. 2025-10-02 ===';
SELECT dt_rep, out_rub_total, rate_avg
FROM   WORK.Forecast_NS_Promo
WHERE  dt_rep BETWEEN '2025-08-29' AND '2025-10-02'
ORDER  BY dt_rep;
GO
