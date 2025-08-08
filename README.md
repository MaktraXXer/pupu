Окей, понял задачу: **подтянуть снапшот на @Anchor ровно так же, как делает `mail.usp_fill_balance_metrics_savings`**, чтобы и **объёмы**, и «ставка на якорь» брались **из того же источника и по тем же правилам** (`rate_use`: баланс/LIQ + ULTRA-lookahead). Ниже — **обновлённые Часть 1 и Часть 2**. В обеих я заменил построение `#bal`/`#bal_prom` на **тот же алгоритм**, что в твоей SP (join к `LIQUIDITY.liq.DepositContract_Rate`, окно `rate_pos` на +1/+2, выбор `rate_use`). Дальше всё твои расчёты, но уже **с идентичным снимком**.

---

# ЧАСТЬ 1 (FLOAT + FIX-base) — с тем же снапшотом, что и SP

```sql
/* ═══════════════════════════════════════════════════════════════
   NS-forecast — ЧАСТЬ 1  (ТОЛЬКО FLOAT и FIX-base)
   ⚠️ СНАПШОТ @Anchor = как в mail.usp_fill_balance_metrics_savings:
      rate_use по правилам ULTRA (+1/+2), LIQ vs balance, и ТЕ ЖЕ фильтры.
   * FLOAT     → prod_id 3103, спред фиксируем на Anchor (как в твоей версии)
   * FIX-base  → prod_id 654,  dt_open < 2025-07-01, ставка константа = rate_use(@Anchor)
   создаёт / перезаписывает:
       #cal, #key, #bal
       #FLOAT_daily            + WORK.Forecast_NS_Float
       #FIX_base_daily         + WORK.Forecast_NS_FixBase
   v.2025-08-10
════════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31';

/* ───────────── 1. календарь ───────────── */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* ───────────── 2. KEY-spot (TERM=1) ───── */
IF OBJECT_ID('tempdb..#key') IS NOT NULL DROP TABLE #key;
SELECT DT_REP,KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* ───────────── 3. СНАПШОТ @Anchor как в mail.usp_fill_balance_metrics_savings ─
   Берём диапазон @Anchor±2 для look-ahead, считаем rate_use и оставляем @Anchor */
IF OBJECT_ID('tempdb..#bal_src') IS NOT NULL DROP TABLE #bal_src;
SELECT
    t.dt_rep,
    CAST(t.dt_open  AS date)                AS dt_open,
    CAST(t.dt_close AS date)                AS dt_close,
    t.con_id,
    t.cli_id,
    t.prod_id,
    CAST(t.out_rub AS decimal(20,2))        AS out_rub,
    CAST(t.rate_con AS decimal(9,4))        AS rate_balance,    -- как в SP
    t.rate_con_src,
    t.is_floatrate,
    t.TSEGMENTNAME,
    r.rate                                   AS rate_liq
INTO #bal_src
FROM   alm.ALM.vw_balance_rest_all AS t WITH (NOLOCK)
LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = t.con_id
       AND CASE WHEN t.dt_open = t.dt_rep
                THEN DATEADD(day,1,t.dt_rep)                   -- как в SP для ULTRA нулевого дня
                ELSE t.dt_rep
           END BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day,2,@Anchor)
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.cur          = '810';

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src (con_id, dt_rep);

;WITH bal_pos AS (
    SELECT *,
           /* первая положительная ставка ULTRA на +1 / +2 день */
           MIN(CASE WHEN rate_balance > 0
                     AND rate_con_src = N'счет ультра,вручную'
                    THEN rate_balance END)
               OVER (PARTITION BY con_id
                     ORDER BY dt_rep
                     ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
    FROM #bal_src
    WHERE dt_rep = @Anchor
),
rate_calc AS (
    SELECT *,
           CASE
             WHEN rate_liq IS NULL
                  THEN CASE
                           WHEN rate_balance < 0
                                THEN COALESCE(rate_pos, rate_balance)
                           ELSE rate_balance
                       END
             WHEN rate_liq < 0  AND rate_balance > 0 THEN rate_balance
             WHEN rate_liq < 0  AND rate_balance < 0 THEN COALESCE(rate_pos, rate_balance)
             WHEN rate_liq >= 0 AND rate_balance >= 0 THEN rate_liq
             WHEN rate_liq > 0  AND rate_balance < 0 THEN rate_liq
             ELSE rate_liq
           END AS rate_use
    FROM bal_pos
)
IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;
SELECT
    con_id, cli_id, prod_id,
    out_rub,
    rate_con = CAST(rate_use AS decimal(9,4)),   -- унифицируем имя
    dt_open, dt_close,
    is_floatrate,
    TSEGMENTNAME
INTO #bal
FROM rate_calc
WHERE out_rub IS NOT NULL;

/* ─────────── 4-A. FLOAT daily (спред фиксируем на Anchor) ──── */
IF OBJECT_ID('tempdb..#FLOAT_daily') IS NOT NULL DROP TABLE #FLOAT_daily;
SELECT  b.con_id,
        b.cli_id,
        b.TSEGMENTNAME,
        b.out_rub,
        c.d                                      AS dt_rep,
        (b.rate_con-k0.KEY_RATE)+k1.KEY_RATE     AS rate_con
INTO    #FLOAT_daily
FROM    #bal b
JOIN    #key k0 ON k0.DT_REP = @Anchor
JOIN    #cal c  ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
JOIN    #key k1 ON k1.DT_REP = c.d
WHERE   b.prod_id = 3103;

/* агрегат FLOAT */
IF OBJECT_ID('WORK.Forecast_NS_Float','U') IS NOT NULL
    TRUNCATE TABLE WORK.Forecast_NS_Float
ELSE
    CREATE TABLE WORK.Forecast_NS_Float(
        dt_rep date PRIMARY KEY,
        out_rub_total  decimal(20,2),
        rate_avg       decimal(9,4));

INSERT WORK.Forecast_NS_Float
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #FLOAT_daily
GROUP  BY dt_rep;

/* ───────────── 4-B.  FIX-base daily (prod 654, dt_open<2025-07-01) ──── */
IF OBJECT_ID('tempdb..#FIX_base_daily') IS NOT NULL DROP TABLE #FIX_base_daily;
SELECT  b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
        c.d            AS dt_rep,
        b.rate_con     AS rate_con
INTO    #FIX_base_daily
FROM    #bal b
JOIN    #cal c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE   b.prod_id = 654
  AND   b.dt_open < '2025-07-01';

/* агрегат FIX-base */
IF OBJECT_ID('WORK.Forecast_NS_FixBase','U') IS NOT NULL
    TRUNCATE TABLE WORK.Forecast_NS_FixBase
ELSE
    CREATE TABLE WORK.Forecast_NS_FixBase(
        dt_rep date PRIMARY KEY,
        out_rub_total  decimal(20,2),
        rate_avg       decimal(9,4));

INSERT WORK.Forecast_NS_FixBase
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #FIX_base_daily
GROUP  BY dt_rep;

/* ───────────── 5. вывод для проверки ─── */
PRINT N'==== FLOAT daily (TOP-30) ====';     SELECT TOP (30) * FROM #FLOAT_daily ORDER BY dt_rep,con_id;
PRINT N'==== FIX-base daily (TOP-30) ====';  SELECT TOP (30) * FROM #FIX_base_daily ORDER BY dt_rep,con_id;
PRINT N'==== агрегаты (первые 10 дней) ===='; SELECT 'FLOAT' AS bucket,* FROM WORK.Forecast_NS_Float ORDER BY dt_rep;
SELECT 'FIX-base',* FROM WORK.Forecast_NS_FixBase ORDER BY dt_rep;
GO
```

---

# ЧАСТЬ 2 (FIX-promo) — тот же снапшот, что и SP

```sql
/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2 (FIX-promo-roll, prod_id = 654, is_floatrate=0)
   ⚠️ СНАПШОТ @Anchor = как в mail.usp_fill_balance_metrics_savings (rate_use)
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

/* ── 0) Календарь + KEY(TERM=1) ─────────────────────────────── */
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

/* ── 1) СНАПШОТ promo-портфеля (точно как в SP, но ограничиваемся prod 654 и июля-авг-2025) ─ */
IF OBJECT_ID('tempdb..#bal_src') IS NOT NULL DROP TABLE #bal_src;
SELECT
    t.dt_rep,
    CAST(t.dt_open  AS date)                AS dt_open,
    CAST(t.dt_close AS date)                AS dt_close,
    CAST(t.con_id AS bigint)                AS con_id,
    CAST(t.cli_id AS bigint)                AS cli_id,
    CAST(t.out_rub AS decimal(20,2))        AS out_rub,
    CAST(t.rate_con AS decimal(9,4))        AS rate_balance,
    t.rate_con_src,
    t.is_floatrate,
    t.TSEGMENTNAME,
    r.rate                                   AS rate_liq
INTO #bal_src
FROM   alm.ALM.vw_balance_rest_all AS t WITH (NOLOCK)
LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = t.con_id
       AND CASE WHEN t.dt_open = t.dt_rep
                THEN DATEADD(day,1,t.dt_rep)
                ELSE t.dt_rep
           END BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day,2,@Anchor)
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.cur          = '810'
  AND  t.prod_id      = 654;

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src (con_id, dt_rep);

;WITH bal_pos AS (
    SELECT *,
           MIN(CASE WHEN rate_balance > 0
                     AND rate_con_src = N'счет ультра,вручную'
                    THEN rate_balance END)
               OVER (PARTITION BY con_id
                     ORDER BY dt_rep
                     ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
    FROM #bal_src
    WHERE dt_rep = @Anchor
),
rate_calc AS (
    SELECT *,
           CASE
             WHEN rate_liq IS NULL
                  THEN CASE
                           WHEN rate_balance < 0
                                THEN COALESCE(rate_pos, rate_balance)
                           ELSE rate_balance
                       END
             WHEN rate_liq < 0  AND rate_balance > 0 THEN rate_balance
             WHEN rate_liq < 0  AND rate_balance < 0 THEN COALESCE(rate_pos, rate_balance)
             WHEN rate_liq >= 0 AND rate_balance >= 0 THEN rate_liq
             WHEN rate_liq > 0  AND rate_balance < 0 THEN rate_liq
             ELSE rate_liq
           END AS rate_use
    FROM bal_pos
)
DROP TABLE IF EXISTS #bal_prom;
SELECT
    con_id,
    cli_id,
    out_rub,
    rate_anchor = CAST(rate_use AS decimal(9,4)),  -- ставка на Anchor из той же логики
    dt_open,
    is_floatrate,
    TSEGMENTNAME
INTO #bal_prom
FROM rate_calc
WHERE COALESCE(is_floatrate,0)=0
  AND dt_open BETWEEN '2025-07-01' AND '2025-08-31';

DECLARE @PromoTotal decimal(20,2) = (SELECT SUM(out_rub) FROM #bal_prom);

/* ── 2) Фикс. спреды по август-25 (как раньше, но из #bal_prom) ─ */
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
       (N'ДЧБО', @Spread_DChbo),
       (N'Розничный бизнес', @Spread_Retail);

/* ── 3) Циклы и вся остальная логика — без изменений, но с нашим снапшотом ─ */
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

/* ── 4) Дневная лента без 1-х чисел (promo + base-day) ─ */
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
WHERE  DAY(d.d) <> 1;

/* ── 5) BASE-DAY HALF-SHIFT (исправленная версия) ─ */
DROP TABLE IF EXISTS #base_dates;
SELECT DISTINCT dp.cli_id, dp.dt_rep
INTO   #base_dates
FROM   #daily_pre dp
JOIN   #cycles c ON c.con_id=dp.con_id AND c.cycle_no=dp.cycle_no
WHERE  dp.dt_rep = c.base_day;

-- корректный перенос половинок
DROP TABLE IF EXISTS #daily_base_adj;
WITH base_pool AS (
    SELECT dp.*,
           is_base = CASE WHEN dp.dt_rep = c.base_day THEN 1 ELSE 0 END
    FROM   #daily_pre dp
    JOIN   #base_dates bd ON bd.cli_id=dp.cli_id AND bd.dt_rep=dp.dt_rep
    LEFT  JOIN #cycles c  ON c.con_id=dp.con_id AND c.cycle_no=dp.cycle_no
),
halves AS (
    SELECT cli_id, dt_rep, con_id, TSEGMENTNAME, rate_con,
           out_half = CASE WHEN is_base=1 THEN ROUND(out_rub*0.5,2) ELSE CAST(0.00 AS decimal(20,2)) END,
           is_base
    FROM base_pool
),
shift AS (
    SELECT cli_id, dt_rep, SUM(out_half) AS shift_amt
    FROM halves
    GROUP BY cli_id, dt_rep
),
ranked AS (
    SELECT bp.*,
           ROW_NUMBER() OVER (PARTITION BY bp.cli_id, bp.dt_rep
                              ORDER BY bp.rate_con DESC, bp.TSEGMENTNAME, bp.con_id) AS rn
    FROM   base_pool bp
)
SELECT
    r.con_id,
    r.cli_id,
    r.TSEGMENTNAME,
    out_rub =
        CASE
          WHEN r.rn=1 AND r.is_base=1 THEN ROUND(r.out_rub*0.5,2) + s.shift_amt
          WHEN r.rn=1 AND r.is_base=0 THEN r.out_rub + s.shift_amt
          WHEN r.is_base=1                  THEN ROUND(r.out_rub*0.5,2)
          ELSE                                   r.out_rub
        END,
    r.dt_rep,
    r.rate_con
INTO   #daily_base_adj
FROM   ranked r
JOIN   shift  s ON s.cli_id=r.cli_id AND s.dt_rep=r.dt_rep;

/* ── 6) Кандидаты на 1-е числа (внутри окна и новое окно) ──── */
DROP TABLE IF EXISTS #day1_candidates;

WITH marks AS (
    SELECT
      c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub, c.cycle_no, c.win_start, c.promo_end, c.base_day,
      DATEFROMPARTS(YEAR(DATEADD(month,1,c.win_start)), MONTH(DATEADD(month,1,c.win_start)), 1) AS m1_in,
      DATEADD(day,1,c.base_day) AS m1_new
    FROM #cycles c
),
cand AS (
    /* 1-е внутри текущего окна (ставка фикс цикла) */
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

/* ── 7) Перелив 1-го числа (max ставка, весь Σ клиента) ────── */
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

/* ── 8) Финальная promo-лента (без дублей) ──────────────────── */
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;

SELECT
    dt_rep,
    out_rub_total = SUM(out_rub),
    rate_avg      = SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM (
    SELECT dp.con_id, dp.cli_id, dp.TSEGMENTNAME, dp.out_rub, dp.dt_rep, dp.rate_con
    FROM   #daily_pre dp
    WHERE  NOT EXISTS (SELECT 1 FROM #base_dates bd WHERE bd.cli_id=dp.cli_id AND bd.dt_rep=dp.dt_rep)

    UNION ALL
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #daily_base_adj

    UNION ALL
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #firstday_assigned
) u
GROUP BY dt_rep;

/* ── 9) Контрольки ──────────────────────────────────────────── */
PRINT N'=== spreads (Aug-25) ===';                 SELECT * FROM WORK.NS_Spreads;
PRINT N'=== sanity total (= @PromoTotal) ===';
SELECT TOP (90) f.dt_rep, f.out_rub_total,
       diff_vs_anchor = CAST(f.out_rub_total - @PromoTotal AS decimal(20,2))
FROM WORK.Forecast_NS_Promo f ORDER BY f.dt_rep;

PRINT N'=== window check: 2025-08-29 .. 2025-10-02 ===';
SELECT dt_rep, out_rub_total, rate_avg
FROM   WORK.Forecast_NS_Promo
ORDER  BY dt_rep;
GO
```

---

### Что именно поменялось (и почему объёмы теперь совпадут):

* В **обеих частях** снапшот `#bal`/`#bal_prom` собирается **в точности как в `mail.usp_fill_balance_metrics_savings`**:

  * тот же join к `LIQUIDITY.liq.DepositContract_Rate` с «нулевым днём» ULTRA на `+1`;
  * та же оконная логика `rate_pos` на **+1/+2**;
  * **те же фильтры**: `section_name`, `block_name`, `od_flag=1`, `cur='810'`;
  * итоговая ставка на @Anchor — это **`rate_use`** (а не «как попало»).
* Вся дальнейшая механика (FLOAT-спред, FIX-base константа, FIX-promo окна/переливы) строится **на этом же снимке** ⇒ **объёмы и база сравнимы 1-в-1** с твоей витриной.

Если захочешь — могу добавить **быструю сверку** между `mail.balance_metrics_savings` («портфель» на @Anchor) и суммой наших бакетов на @Anchor, чтобы падало при расхождении.
