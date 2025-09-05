ок, давай финализируем Часть 2 в том виде, как ты сейчас хочешь:

* ставка по дням моделируется по правилам окна: промо с даты открытия до предпоследнего дня «следующего» месяца; в **последний день месяца** у тех, у кого это `base_day`, ставка = `@BaseRate`, у остальных — их промо;
* **переложение объёмов = ПОЛНЫЙ перелив Σ клиента**:

  * на `base_day` выбираем у клиента строку с **макс. дневной ставкой** и кладём на неё весь Σ (ставка = ставка победителя в этот день, не затираем базовой);
  * на **1-е число** также выбираем по **макс. ставке** из кандидатов (внутри окна и новое окно), на победителя кладём весь Σ клиента.
* спреды считаем **от входных промо-ставок** к `KEY(@Anchor)` (как на скрине).
* снапшот портфеля на `@Anchor` — **в точности как в твоём SP** (ULTRA +1/+2 и LIQ vs balance).

Вот рабочий монолит (T-SQL) с этими правилами:

```sql
/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2 (FIX-promo-roll, prod_id = 654)
   ⚑ СНАПШОТ @Anchor: как в mail.usp_fill_balance_metrics_savings
      (ULTRA +1/+2; LIQ vs balance; те же фильтры).
   ИЗМЕНЕНИЯ:
     (A) Промо-ставки задаём входом и превращаем в спреды к KEY(@Anchor)
     (B) На последний день месяца выполняем ПОЛНЫЙ перелив (Σ клиента)
         на счёт с максимальной ставкой; 1-го числа — тоже ПОЛНЫЙ перелив.
   v.2025-08-11 — full shift on base-day + full shift on 1st; no dupes
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
SET NOCOUNT ON;

/* ── параметры ─────────────────────────────────────────────── */
DECLARE
    @Scenario   tinyint      = 1,
    @Anchor     date         = '2025-09-03',  -- пример
    @HorizonTo  date         = '2025-12-31',
    @BaseRate   decimal(9,4) = 0.0450;       -- базовая ставка на base-day

/* ⌚ диапазон dt_open: "предыдущий месяц .. конец месяца Anchor" */
DECLARE
    @OpenFrom date = DATEFROMPARTS(YEAR(DATEADD(month,-1,@Anchor)), MONTH(DATEADD(month,-1,@Anchor)), 1),
    @OpenTo   date = EOMONTH(@Anchor);

/* ── 0) Календарь горизонта + KEY(TERM=1) ───────────────────── */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d = @Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

IF OBJECT_ID('tempdb..#key') IS NOT NULL DROP TABLE #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario = @Scenario AND TERM = 1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

DECLARE @KeyAtAnchor decimal(9,4);
SELECT @KeyAtAnchor = KEY_RATE FROM #key WHERE DT_REP = @Anchor;

/* ── (A) Входной справочник промо-ставок → спреды к KEY(@Anchor) ─ */
DECLARE
    @PromoRate_DChbo  decimal(9,4) = 0.1660,  -- пример: ДЧБО
    @PromoRate_Retail decimal(9,4) = 0.1640;  -- пример: Розница

DECLARE
    @Spread_DChbo  decimal(9,4) = @PromoRate_DChbo  - @KeyAtAnchor,
    @Spread_Retail decimal(9,4) = @PromoRate_Retail - @KeyAtAnchor;

/* (необязательно) витрина спредов для контроля */
IF OBJECT_ID('WORK.NS_Spreads','U') IS NOT NULL DROP TABLE WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(
  TSEGMENTNAME nvarchar(40) PRIMARY KEY,
  spread       decimal(9,4) NOT NULL
);
INSERT WORK.NS_Spreads(TSEGMENTNAME,spread) VALUES
(N'ДЧБО',            @Spread_DChbo),
(N'Розничный бизнес',@Spread_Retail);

/* ── 1) СНАПШОТ promo-портфеля на @Anchor (ровно как в SP) ─── */
IF OBJECT_ID('tempdb..#bal_src') IS NOT NULL DROP TABLE #bal_src;
SELECT
    t.dt_rep,
    CAST(t.dt_open  AS date)          AS dt_open,
    CAST(t.dt_close AS date)          AS dt_close,
    CAST(t.con_id   AS bigint)        AS con_id,
    CAST(t.cli_id   AS bigint)        AS cli_id,
    CAST(t.prod_id  AS int)           AS prod_id,
    CAST(t.out_rub  AS decimal(20,2)) AS out_rub,
    CAST(t.rate_con AS decimal(9,4))  AS rate_balance,
    t.rate_con_src,
    t.TSEGMENTNAME,
    CAST(r.rate     AS decimal(9,4))  AS rate_liq
INTO #bal_src
FROM   alm.ALM.vw_balance_rest_all AS t WITH (NOLOCK)
LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = t.con_id
       AND CASE WHEN t.dt_open = t.dt_rep
                THEN DATEADD(day,1,t.dt_rep)      -- ULTRA «нулевой» день → +1
                ELSE t.dt_rep
           END BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day,2,@Anchor)
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.cur          = '810'
  AND  t.prod_id      = 654;                      -- промо-продукт

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src (con_id, dt_rep);

IF OBJECT_ID('tempdb..#bal_prom') IS NOT NULL DROP TABLE #bal_prom;
;WITH bal_pos AS (
    SELECT *,
           /* первая положительная ULTRA-ставка на +1/+2 день */
           MIN(CASE WHEN rate_balance > 0
                     AND rate_con_src = N'счет ультра,вручную'
                    THEN rate_balance END)
               OVER (PARTITION BY con_id ORDER BY dt_rep
                     ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
    FROM #bal_src
    WHERE dt_rep = @Anchor
),
rate_calc AS (
    SELECT *,
           CASE
             WHEN rate_liq IS NULL THEN
                  CASE WHEN rate_balance < 0 THEN COALESCE(rate_pos, rate_balance)
                       ELSE rate_balance END
             WHEN rate_liq < 0  AND rate_balance > 0 THEN rate_balance
             WHEN rate_liq < 0  AND rate_balance < 0 THEN COALESCE(rate_pos, rate_balance)
             WHEN rate_liq >= 0 AND rate_balance >= 0 THEN rate_liq
             WHEN rate_liq > 0  AND rate_balance < 0 THEN rate_liq
             ELSE rate_liq
           END AS rate_use
    FROM bal_pos
)
SELECT
    con_id,
    cli_id,
    out_rub,
    rate_anchor = CAST(rate_use AS decimal(9,4)),  -- ставка на Anchor из той же логики
    dt_open,
    TSEGMENTNAME
INTO #bal_prom
FROM rate_calc
WHERE out_rub IS NOT NULL
  AND dt_open BETWEEN @OpenFrom AND @OpenTo;

DECLARE @PromoTotal decimal(20,2) = (SELECT SUM(out_rub) FROM #bal_prom);

/* ── 2) Циклы (base_day = EOM(next), promo_end = base_day-1) ── */
IF OBJECT_ID('tempdb..#cycles') IS NOT NULL DROP TABLE #cycles;
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

/* ── 3) Дневная лента без 1-х чисел (promo + base-day) ──────── */
IF OBJECT_ID('tempdb..#daily_pre') IS NOT NULL DROP TABLE #daily_pre;
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

/* ── 4) BASE-DAY FULL SHIFT: Σ клиента → счёт с максимальной ставкой ─ */
IF OBJECT_ID('tempdb..#base_dates') IS NOT NULL DROP TABLE #base_dates;
SELECT DISTINCT dp.cli_id, dp.dt_rep
INTO   #base_dates
FROM   #daily_pre dp
JOIN   #cycles c ON c.con_id=dp.con_id AND c.cycle_no=dp.cycle_no
WHERE  dp.dt_rep = c.base_day;  -- базовый день есть у клиента

IF OBJECT_ID('tempdb..#daily_base_adj') IS NOT NULL DROP TABLE #daily_base_adj;
;WITH base_pool AS (
    SELECT dp.*
    FROM   #daily_pre dp
    JOIN   #base_dates bd ON bd.cli_id=dp.cli_id AND bd.dt_rep=dp.dt_rep
),
ranked AS (
    SELECT bp.*,
           ROW_NUMBER() OVER (PARTITION BY bp.cli_id, bp.dt_rep
                              ORDER BY bp.rate_con DESC, bp.TSEGMENTNAME, bp.con_id) AS rn
    FROM   base_pool bp
),
sums AS (
    SELECT cli_id, dt_rep, SUM(out_rub) AS total_out
    FROM   base_pool
    GROUP  BY cli_id, dt_rep
)
SELECT
    r.con_id,
    r.cli_id,
    r.TSEGMENTNAME,
    out_rub  = CASE WHEN r.rn=1 THEN s.total_out ELSE CAST(0.00 AS decimal(20,2)) END,
    r.dt_rep,
    rate_con = r.rate_con   -- ставка победителя (может быть промо, может базовая)
INTO   #daily_base_adj
FROM   ranked r
JOIN   sums   s ON s.cli_id=r.cli_id AND s.dt_rep=r.dt_rep;

/* ── 5) 1-е числа: кандидаты (внутри окна и новое окно) ────── */
IF OBJECT_ID('tempdb..#day1_candidates') IS NOT NULL DROP TABLE #day1_candidates;
;WITH marks AS (
    SELECT
      c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub, c.cycle_no, c.win_start, c.promo_end, c.base_day,
      DATEFROMPARTS(YEAR(DATEADD(month,1,c.win_start)), MONTH(DATEADD(month,1,c.win_start)), 1) AS m1_in,
      DATEADD(day,1,c.base_day) AS m1_new
    FROM #cycles c
),
cand AS (
    /* 1-е внутри текущего окна — ставка фикс цикла */
    SELECT
        m.con_id, m.cli_id, m.TSEGMENTNAME, m.out_rub, m.cycle_no,
        dt_rep   = m.m1_in,
        rate_con =
            CASE WHEN m.cycle_no=0
                 THEN bp.rate_anchor
                 ELSE (CASE m.TSEGMENTNAME
                        WHEN N'ДЧБО'            THEN @Spread_DChbo  + k_open.KEY_RATE
                        ELSE                          @Spread_Retail+ k_open.KEY_RATE
                      END)
            END
    FROM marks m
    JOIN #bal_prom bp      ON bp.con_id      = m.con_id
    LEFT JOIN #key k_open  ON k_open.DT_REP  = m.win_start
    WHERE m.m1_in BETWEEN m.win_start AND m.promo_end
      AND m.m1_in BETWEEN @Anchor   AND @HorizonTo

    UNION ALL

    /* 1-е нового окна — новая промо (KEY(1-е)+спред) */
    SELECT
        m.con_id, m.cli_id, m.TSEGMENTNAME, m.out_rub, m.cycle_no + 1 AS cycle_no,
        dt_rep   = m.m1_new,
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
IF OBJECT_ID('tempdb..#cli_sum') IS NOT NULL DROP TABLE #cli_sum;
SELECT cli_id, SUM(out_rub) AS out_rub_sum
INTO   #cli_sum
FROM   #bal_prom
GROUP  BY cli_id;

/* ── 6) 1-е числа: ПОЛНЫЙ перелив Σ клиента на max ставку ──── */
IF OBJECT_ID('tempdb..#firstday_assigned') IS NOT NULL DROP TABLE #firstday_assigned;
;WITH ranked AS (
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
    out_rub       = cs.out_rub_sum,        -- весь объём клиента
    r.dt_rep,
    rate_con      = MAX(CASE WHEN rn=1 THEN rate_con END)
INTO   #firstday_assigned
FROM   ranked r
JOIN   #cli_sum cs ON cs.cli_id = r.cli_id
GROUP  BY r.cli_id, r.dt_rep, cs.out_rub_sum;

/* ── 7) Финальная promo-лента (заменяем base-day и 1-е числа) ─ */
IF OBJECT_ID('WORK.Forecast_NS_Promo','U') IS NOT NULL DROP TABLE WORK.Forecast_NS_Promo;

SELECT
    dt_rep,
    out_rub_total = SUM(out_rub),
    rate_avg      = SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM (
    /* все дни, кроме base-day у отмеченных клиентов (их заменим) */
    SELECT dp.con_id, dp.cli_id, dp.TSEGMENTNAME, dp.out_rub, dp.dt_rep, dp.rate_con
    FROM   #daily_pre dp
    WHERE  NOT EXISTS (
           SELECT 1 FROM #base_dates bd
           WHERE  bd.cli_id=dp.cli_id AND bd.dt_rep=dp.dt_rep)

    UNION ALL
    /* base-day — full shift (одна строка на клиента) */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #daily_base_adj

    UNION ALL
    /* 1-е числа — full shift (одна строка на клиента) */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #firstday_assigned
) u
GROUP BY dt_rep;

/* ── 8) Контрольки ─────────────────────────────────────────── */
PRINT N'=== spreads at Anchor ===';  SELECT * FROM WORK.NS_Spreads;

PRINT N'=== sanity: Σ объёма на каждый день (должен равняться Σ на Anchor) ===';
SELECT TOP (120) f.dt_rep, f.out_rub_total,
       diff_vs_anchor = CAST(f.out_rub_total - @PromoTotal AS decimal(20,2))
FROM WORK.Forecast_NS_Promo f
ORDER BY f.dt_rep;

PRINT N'=== окно: последние 5 дней месяца и первые 3 следующего (для проверки перелива) ===';
;WITH wnd AS (
  SELECT d FROM #cal WHERE DAY(d) >= 26
  UNION ALL SELECT DATEADD(day,1,EOMONTH(@Anchor))
  UNION ALL SELECT DATEADD(day,2,EOMONTH(@Anchor))
  UNION ALL SELECT DATEADD(day,3,EOMONTH(@Anchor))
)
SELECT p.*
FROM WORK.Forecast_NS_Promo p
JOIN wnd w ON w.d = p.dt_rep
ORDER BY p.dt_rep;
GO
```

Это ровно то, что ты просил: сначала «моделируем ставку по дням» (правило 2-месячного промо → base-day → новое окно), затем **на base-day и 1-го числа** делаем **полный перелив Σ клиента** на «лучший» счёт по ставке. Объёмы не теряются: `Σ(out_rub)` по каждому дню равен `Σ` на `@Anchor` (см. sanity-блок).
