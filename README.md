/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2 (FIX-promo-roll, prod_id = 654)
   Снапшот @Anchor как в mail.usp_fill_balance_metrics_savings
   + ДОБАВЛЕНЫ нулевые договоры ALM.ehd.import_ZeroContracts
     (с учётом TSEGMENTNAME и подстановкой промо-ставок).
 
   ФЛАГИ:
   • @UseFixedPromoFromNextMonth: ВКЛ → фикс-промо (@PromoRate_*)
     действует ТОЛЬКО в месяце, следующем за якорем:
     окно [ @FixedPromoStart ; @FixedPromoEnd ).
   • @UseHardPromoSpread: ВКЛ → вместо расчетного спреда использовать
     жёстко заданный спред (@HardSpread_*), начиная с @HardSpreadStart.
   ПРИОРИТЕТ: фикс-промо (если окно)  → иначе жесткий спред (если включён)
             → иначе стандартная модель (как была).
 
   Правила:
     • промо: с dt_open до предпоследнего дня следующего месяца;
     • base_day — базовая @BaseRate;
     • base_day и 1-е числа — ПОЛНЫЙ перелив Σ клиента на max ставку;
     • НОВОЕ: после 1-го числа клиент «залипает» на выбранной max-ставке
       до promo_end (carry-forward).
   v.2025-08-11 (+флаги + carry-forward)
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
SET NOCOUNT ON;

/* ── параметры ─────────────────────────────────────────────── */
DECLARE
    @Scenario   tinyint      = 1,
    @Anchor     date         = '2025-10-27',
    @HorizonTo  date         = '2025-12-31',
    @BaseRate   decimal(9,4) = 0.0450,  -- базовая ставка на base-day

    /* Флаг 1: фикс-промо ТОЛЬКО в месяце, следующем за якорем */
    @UseFixedPromoFromNextMonth bit = 0,

    /* Флаг 2: жёсткие спреды (заменяют расчетные спреды) */
    @UseHardPromoSpread bit = 0;

/* Окно фикс-промо: ровно один месяц после якоря */
DECLARE @FixedPromoStart date = DATEADD(day,1,EOMONTH(@Anchor));   -- 1-е следующего месяца
DECLARE @FixedPromoEnd   date = DATEADD(month,1,@FixedPromoStart); -- 1-е через месяц

/* Старт применения жёстких спредов (по умолчанию — с якоря; можно переопределить) */
DECLARE @HardSpreadStart date = @Anchor;

/* диапазон dt_open для «живых» договоров: пред. месяц .. EOM(Anchor) */
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

/* ── (A) Промо-ставки и расчетные спреды (старая логика) ───── */
DECLARE
    @PromoRate_DChbo  decimal(9,4) = 0.1600,  -- промо ДЧБО (УЧК)
    @PromoRate_Retail decimal(9,4) = 0.1590;  -- промо Розница;

DECLARE
    @Spread_DChbo  decimal(9,4) = @PromoRate_DChbo  - @KeyAtAnchor,
    @Spread_Retail decimal(9,4) = @PromoRate_Retail - @KeyAtAnchor;

/* ── (A2) Жёсткие спреды (новая опция) ─────────────────────── */
DECLARE
    @HardSpread_DChbo  decimal(9,4) = -0.0030,  -- напр. +0.30 п.п. к KEY(win_start) → спред -0.0030 к KEY(@Anchor) для примера
    @HardSpread_Retail decimal(9,4) = -0.0040;  -- напр. +0.40 п.п.

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
    CAST(t.dt_open  AS date)                 AS dt_open,
    CAST(t.dt_close AS date)                 AS dt_close,
    CAST(t.con_id   AS bigint)               AS con_id,
    CAST(t.cli_id   AS bigint)               AS cli_id,
    CAST(t.prod_id  AS int)                  AS prod_id,
    CAST(t.out_rub  AS decimal(20,2))        AS out_rub,
    CAST(t.rate_con AS decimal(9,4))         AS rate_balance,
    t.rate_con_src,
    t.TSEGMENTNAME,
    CAST(r.rate     AS decimal(9,4))         AS rate_liq
INTO #bal_src
FROM   alm.ALM.vw_balance_rest_all AS t WITH (NOLOCK)
LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = t.con_id
       AND CASE WHEN t.dt_open = t.dt_rep
                THEN DATEADD(day,1,t.dt_rep)        -- ULTRA «нулевой» день → +1
                ELSE t.dt_rep
           END BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day,2,@Anchor)
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.cur          = '810'
  AND  t.prod_id      = 654;

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src (con_id, dt_rep);

IF OBJECT_ID('tempdb..#bal_prom') IS NOT NULL DROP TABLE #bal_prom;
;WITH bal_pos AS (
    SELECT *,
           /* первая положительная ULTRA-ставка на +1/+2 день */
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
SELECT
    con_id,
    cli_id,
    out_rub,
    rate_anchor = CAST(rate_use AS decimal(9,4)),  -- ставка на Anchor
    dt_open,
    TSEGMENTNAME
INTO #bal_prom
FROM rate_calc
WHERE out_rub IS NOT NULL
  AND dt_open BETWEEN @OpenFrom AND @OpenTo;

/* ── 1a) ДОБАВЛЯЕМ НУЛЕВЫЕ ДОГОВОРЫ (dt_open <= @Anchor) ───── */
IF OBJECT_ID('tempdb..#z0') IS NOT NULL DROP TABLE #z0;
SELECT
    CAST(z.con_id AS bigint)                                                    AS con_id,
    CAST(z.cli_id AS bigint)                                                    AS cli_id,
    CAST(0.00     AS decimal(20,2))                                             AS out_rub,
    CAST(
         COALESCE(
             z.rate_con,
             CASE WHEN LTRIM(RTRIM(COALESCE(z.TSEGMENTNAME,N''))) = N'ДЧБО'
                  THEN @PromoRate_DChbo
                  ELSE @PromoRate_Retail
             END
         )
         AS decimal(9,4)
    )                                                                           AS rate_anchor,
    CAST(z.dt_open AS date)                                                     AS dt_open,
    CASE WHEN LTRIM(RTRIM(COALESCE(z.TSEGMENTNAME,N''))) = N''
         THEN N'Розничный бизнес'
         ELSE z.TSEGMENTNAME
    END                                                                          AS TSEGMENTNAME
INTO #z0
FROM ALM.ehd.import_ZeroContracts z WITH (NOLOCK)
WHERE z.dt_open <= @Anchor
  AND NOT EXISTS (SELECT 1 FROM #bal_prom p WHERE p.con_id = z.con_id);

INSERT #bal_prom (con_id, cli_id, out_rub, rate_anchor, dt_open, TSEGMENTNAME)
SELECT con_id, cli_id, out_rub, rate_anchor, dt_open, TSEGMENTNAME
FROM #z0;

/* контроль общего объёма на Anchor (живые объёмы; нулевые = 0) */
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
           /* ПРИОРИТЕТ 1: Фикс-промо только в окне следующего месяца */
           CASE
             WHEN @UseFixedPromoFromNextMonth = 1
              AND d.d >= @FixedPromoStart AND d.d < @FixedPromoEnd THEN
                 CASE c.TSEGMENTNAME
                       WHEN N'ДЧБО'            THEN @PromoRate_DChbo
                       ELSE                          @PromoRate_Retail
                  END
             /* ПРИОРИТЕТ 2: Жёсткие спреды (замена расчетных спредов) */
             WHEN @UseHardPromoSpread = 1 AND d.d >= @HardSpreadStart THEN
                  CASE
                    WHEN c.cycle_no = 0
                         THEN bp.rate_anchor  -- нулевой цикл как в модели
                    ELSE CASE c.TSEGMENTNAME
                           WHEN N'ДЧБО'            THEN @HardSpread_DChbo  + k_open.KEY_RATE
                           ELSE                          @HardSpread_Retail+ k_open.KEY_RATE
                         END
                  END
             /* Иначе: исходная модель (как была) */
             ELSE
                  CASE
                    WHEN c.cycle_no = 0
                         THEN bp.rate_anchor
                    ELSE CASE c.TSEGMENTNAME
                           WHEN N'ДЧБО'            THEN @Spread_DChbo  + k_open.KEY_RATE
                           ELSE                          @Spread_Retail+ k_open.KEY_RATE
                         END
                  END
           END
      WHEN d.d = c.base_day THEN @BaseRate
    END AS rate_con
INTO   #daily_pre
FROM   #cycles c
JOIN   #cal d       ON d.d BETWEEN c.win_start AND c.base_day
LEFT   JOIN #key k_open ON k_open.DT_REP = c.win_start
JOIN   #bal_prom bp ON bp.con_id = c.con_id
WHERE  DAY(d.d) <> 1;   -- 1-е числа делаем отдельно

/* ── 4) base_day — ПОЛНЫЙ перелив Σ клиента на max ставку дня ─ */
IF OBJECT_ID('tempdb..#base_dates') IS NOT NULL DROP TABLE #base_dates;
SELECT DISTINCT dp.cli_id, dp.dt_rep
INTO   #base_dates
FROM   #daily_pre dp
JOIN   #cycles c ON c.con_id=dp.con_id AND c.cycle_no=dp.cycle_no
WHERE  dp.dt_rep = c.base_day;

IF OBJECT_ID('tempdb..#daily_base_adj') IS NOT NULL DROP TABLE #daily_base_adj;
;WITH pool AS (
    SELECT dp.*
    FROM   #daily_pre dp
    JOIN   #base_dates bd ON bd.cli_id=dp.cli_id AND bd.dt_rep=dp.dt_rep
),
ranked AS (
    SELECT p.*,
           ROW_NUMBER() OVER (PARTITION BY p.cli_id, p.dt_rep
                              ORDER BY p.rate_con DESC, p.TSEGMENTNAME, p.con_id) AS rn
    FROM   pool p
),
sums AS (
    SELECT cli_id, dt_rep, SUM(out_rub) AS total_out
    FROM   pool
    GROUP  BY cli_id, dt_rep
)
SELECT
    r.con_id,
    r.cli_id,
    r.TSEGMENTNAME,
    out_rub  = CASE WHEN r.rn=1 THEN s.total_out ELSE CAST(0.00 AS decimal(20,2)) END,
    r.dt_rep,
    rate_con = r.rate_con
INTO   #daily_base_adj
FROM   ranked r
JOIN   sums   s ON s.cli_id=r.cli_id AND s.dt_rep=r.dt_rep;

/* ── 5) 1-е числа — кандидаты (внутри окна и новое окно) ───── */
IF OBJECT_ID('tempdb..#day1_candidates') IS NOT NULL DROP TABLE #day1_candidates;
;WITH marks AS (
    SELECT
      c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub, c.cycle_no, c.win_start, c.promo_end, c.base_day,
      DATEFROMPARTS(YEAR(DATEADD(month,1,c.win_start)), MONTH(DATEADD(month,1,c.win_start)), 1) AS m1_in,
      DATEADD(day,1,c.base_day) AS m1_new
    FROM #cycles c
),
cand AS (
    /* 1-е внутри текущего окна */
    SELECT
        m.con_id, m.cli_id, m.TSEGMENTNAME, m.out_rub, m.cycle_no,
        dt_rep = m.m1_in,
        rate_con =
            CASE
              WHEN @UseFixedPromoFromNextMonth = 1
               AND m.m1_in >= @FixedPromoStart AND m.m1_in < @FixedPromoEnd THEN
                   CASE m.TSEGMENTNAME
                        WHEN N'ДЧБО'            THEN @PromoRate_DChbo
                        ELSE                          @PromoRate_Retail
                   END
              WHEN @UseHardPromoSpread = 1 AND m.m1_in >= @HardSpreadStart THEN
                   CASE
                     WHEN m.cycle_no = 0
                          THEN bp.rate_anchor
                     ELSE CASE m.TSEGMENTNAME
                            WHEN N'ДЧБО'            THEN @HardSpread_DChbo  + k_open.KEY_RATE
                            ELSE                          @HardSpread_Retail+ k_open.KEY_RATE
                          END
                   END
              ELSE
                   CASE
                     WHEN m.cycle_no = 0
                          THEN bp.rate_anchor
                     ELSE CASE m.TSEGMENTNAME
                            WHEN N'ДЧБО'            THEN @Spread_DChbo  + k_open.KEY_RATE
                            ELSE                          @Spread_Retail+ k_open.KEY_RATE
                          END
                   END
            END
    FROM marks m
    JOIN #bal_prom bp ON bp.con_id = m.con_id
    LEFT JOIN #key k_open ON k_open.DT_REP = m.win_start
    WHERE m.m1_in BETWEEN m.win_start AND m.promo_end
      AND m.m1_in BETWEEN @Anchor AND @HorizonTo

    UNION ALL

    /* 1-е нового окна */
    SELECT
        m.con_id, m.cli_id, m.TSEGMENTNAME, m.out_rub, m.cycle_no + 1 AS cycle_no,
        dt_rep = m.m1_new,
        rate_con =
            CASE
              WHEN @UseFixedPromoFromNextMonth = 1
               AND m.m1_new >= @FixedPromoStart AND m.m1_new < @FixedPromoEnd THEN
                   CASE m.TSEGMENTNAME
                        WHEN N'ДЧБО'            THEN @PromoRate_DChbo
                        ELSE                          @PromoRate_Retail
                   END
              WHEN @UseHardPromoSpread = 1 AND m.m1_new >= @HardSpreadStart THEN
                   CASE m.TSEGMENTNAME
                     WHEN N'ДЧБО'            THEN @HardSpread_DChbo  + k_new.KEY_RATE
                     ELSE                          @HardSpread_Retail+ k_new.KEY_RATE
                   END
              ELSE
                   CASE m.TSEGMENTNAME
                     WHEN N'ДЧБО'            THEN @Spread_DChbo  + k_new.KEY_RATE
                     ELSE                          @Spread_Retail+ k_new.KEY_RATE
                   END
            END
    FROM marks m
    LEFT JOIN #key k_new ON k_new.DT_REP = m.m1_new
    WHERE m.m1_new BETWEEN @Anchor AND @HorizonTo
)
SELECT * INTO #day1_candidates FROM cand;

/* Σ-объём клиента на Anchor — для 1-х чисел (полный перелив) ─ */
IF OBJECT_ID('tempdb..#cli_sum') IS NOT NULL DROP TABLE #cli_sum;
SELECT cli_id, SUM(out_rub) AS out_rub_sum
INTO   #cli_sum
FROM   #bal_prom
GROUP  BY cli_id;

/* ── 6) 1-е числа — ПОЛНЫЙ перелив Σ клиента на max ставку ─── */
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

/* ── 6.5) Carry-forward после 1-го числа: клиент остаётся на max-ставке до promo_end ─ */
IF OBJECT_ID('tempdb..#carry_forward') IS NOT NULL DROP TABLE #carry_forward;
;WITH pick AS (
    /* что выбрали 1-го числа: con_id, сегмент, дата перелива */
    SELECT f.cli_id, f.con_id, f.TSEGMENTNAME, f.dt_rep AS first_day
    FROM   #firstday_assigned f
),
bind AS (
    /* к какому окну (cycle) относится это 1-е число */
    SELECT p.cli_id, p.con_id, p.TSEGMENTNAME, p.first_day,
           c.cycle_no, c.win_start, c.promo_end, c.base_day
    FROM   pick p
    JOIN   #cycles c
           ON c.con_id = p.con_id
          AND p.first_day BETWEEN c.win_start AND c.base_day
),
days AS (
    /* дни после 1-го числа до конца promo-окна включительно */
    SELECT b.cli_id, b.con_id, b.TSEGMENTNAME, b.first_day,
           b.cycle_no, b.win_start, b.promo_end,
           d.d AS dt_rep
    FROM   bind b
    JOIN   #cal d
           ON d.d BETWEEN DATEADD(day,1,b.first_day) AND b.promo_end
),
kref AS (
    /* KEY на старт окна (k_open = KEY(win_start)) */
    SELECT DISTINCT b.win_start, k.KEY_RATE AS key_open
    FROM   bind b
    JOIN   #key k ON k.DT_REP = b.win_start
)
SELECT
    cf.con_id,
    cf.cli_id,
    cf.TSEGMENTNAME,
    /* весь клиентский объём: тот же Σ, что и на 1-е число */
    out_rub = cs.out_rub_sum,
    cf.dt_rep,
    /* ставка по выбранному сегменту, с приоритетами:
       1) фикс-промо в месяце после якоря
       2) жёсткий спред (если включён)
       3) обычный спред от промо (@Spread_*)
       все спреды считаются от KEY(win_start) окна */
    rate_con =
      CASE
        WHEN @UseFixedPromoFromNextMonth = 1
         AND cf.dt_rep >= @FixedPromoStart AND cf.dt_rep < @FixedPromoEnd THEN
             CASE cf.TSEGMENTNAME
               WHEN N'ДЧБО'            THEN @PromoRate_DChbo
               ELSE                          @PromoRate_Retail
             END
        WHEN @UseHardPromoSpread = 1 AND cf.dt_rep >= @HardSpreadStart THEN
             CASE cf.TSEGMENTNAME
               WHEN N'ДЧБО'            THEN @HardSpread_DChbo  + kr.key_open
               ELSE                          @HardSpread_Retail+ kr.key_open
             END
        ELSE
             CASE cf.TSEGMENTNAME
               WHEN N'ДЧБО'            THEN @Spread_DChbo  + kr.key_open
               ELSE                          @Spread_Retail+ kr.key_open
             END
      END
INTO   #carry_forward
FROM   days cf
JOIN   kref kr  ON kr.win_start = cf.win_start
JOIN   #cli_sum cs ON cs.cli_id  = cf.cli_id;

/* ── 7) Финальная promo-лента с учётом:
       — base_day (full-shift, как было),
       — 1-е числа (full-shift, как было),
       — carry-forward после 1-го числа до promo_end ─────────── */
IF OBJECT_ID('WORK.Forecast_NS_Promo','U') IS NOT NULL DROP TABLE WORK.Forecast_NS_Promo;

SELECT
    dt_rep,
    out_rub_total = SUM(out_rub),
    rate_avg      = SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM (
    /* 7.1. Все дни из #daily_pre, КРОМЕ:
       — base_day у отмеченных клиентов (заменим),
       — 1-х чисел (заменим),
       — дней после 1-го до promo_end для клиентов с carry-forward (заменим) */
    SELECT dp.con_id, dp.cli_id, dp.TSEGMENTNAME, dp.out_rub, dp.dt_rep, dp.rate_con
    FROM   #daily_pre dp
    WHERE  NOT EXISTS (SELECT 1 FROM #base_dates bd
                       WHERE bd.cli_id = dp.cli_id AND bd.dt_rep = dp.dt_rep)
       AND NOT EXISTS (SELECT 1 FROM #firstday_assigned fa
                       WHERE fa.cli_id = dp.cli_id AND fa.dt_rep = dp.dt_rep)
       AND NOT EXISTS (SELECT 1 FROM #carry_forward cf
                       WHERE cf.cli_id = dp.cli_id AND cf.dt_rep = dp.dt_rep)

    UNION ALL
    /* 7.2. base_day — full shift (одна строка на клиента) */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #daily_base_adj

    UNION ALL
    /* 7.3. 1-е числа — full shift (одна строка на клиента) */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #firstday_assigned

    UNION ALL
    /* 7.4. carry-forward: со 2-го числа до promo_end сидим на выбранной max-ставке */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #carry_forward
) u
GROUP BY dt_rep;

/* ── 8) Контрольки ─────────────────────────────────────────── */
PRINT N'=== spreads at Anchor ===';  SELECT * FROM WORK.NS_Spreads;

PRINT N'=== sanity: Σ объёма по дням (равен Σ на Anchor) ===';
SELECT TOP (200) f.dt_rep, f.out_rub_total,
       diff_vs_anchor = CAST(f.out_rub_total - @PromoTotal AS decimal(20,2)),
       f.rate_avg
FROM WORK.Forecast_NS_Promo f
ORDER BY f.dt_rep;
GO

SELECT * FROM WORK.Forecast_NS_Promo f
ORDER BY f.dt_rep;
