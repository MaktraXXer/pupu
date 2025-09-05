/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2 (FIX-promo-roll, prod_id = 654) — v2
   ⚑ СНАПШОТ @Anchor: как в mail.usp_fill_balance_metrics_savings
      (ULTRA +1/+2; LIQ vs balance; те же фильтры).
   ⚑ Промо-ставки задаются входом → спреды к KEY(@Anchor).
   ⚑ Каждому клиенту с его ПЕРВОГО base-day (min по окнам) и далее
     КАЖДЫЙ месяц открываем НОВЫЙ счёт с промо на 70% Σ клиента.
     Старые счета масштабируются (1-0.7)^{N(t)} — объём не теряется.
   Вывод: WORK.Forecast_NS_Promo (дневные агрегаты)
   v.2025-08-12
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
SET NOCOUNT ON;

/* ── параметры ─────────────────────────────────────────────── */
DECLARE
    @Scenario   tinyint      = 1,
    @Anchor     date         = '2025-09-03',
    @HorizonTo  date         = '2025-12-31',
    @BaseRate   decimal(9,4) = 0.0450,      -- базовая ставка
    @ShareNew   decimal(9,6) = 0.700000;    -- 70% в новые счета

/* охватить открытия: месяц ДО Anchor + месяц Anchor */
DECLARE
    @OpenFrom date = DATEFROMPARTS(YEAR(DATEADD(month,-1,@Anchor)), MONTH(DATEADD(month,-1,@Anchor)), 1),
    @OpenTo   date = EOMONTH(@Anchor);

/* ── 0) Календарь горизонта + KEY(TERM=1) ───────────────────── */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

IF OBJECT_ID('tempdb..#key') IS NOT NULL DROP TABLE #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@Scenario AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

DECLARE @KeyAtAnchor decimal(9,4);
SELECT @KeyAtAnchor = KEY_RATE FROM #key WHERE DT_REP=@Anchor;

/* ── (A) Входной справочник промо-ставок → спреды к KEY(@Anchor) ─ */
DECLARE
    @PromoRate_DChbo  decimal(9,4) = 0.1660,  -- ДЧБО (вход)
    @PromoRate_Retail decimal(9,4) = 0.1640;  -- Розничный бизнес (вход)

DECLARE
    @Spread_DChbo  decimal(9,4) = @PromoRate_DChbo  - @KeyAtAnchor,
    @Spread_Retail decimal(9,4) = @PromoRate_Retail - @KeyAtAnchor;

IF OBJECT_ID('WORK.NS_Spreads','U') IS NOT NULL DROP TABLE WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(
  TSEGMENTNAME nvarchar(40) PRIMARY KEY,
  spread       decimal(9,4) NOT NULL
);
INSERT WORK.NS_Spreads VALUES (N'ДЧБО',@Spread_DChbo),(N'Розничный бизнес',@Spread_Retail);

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
    rate_anchor = CAST(rate_use AS decimal(9,4)),  -- ставка на Anchor
    dt_open,
    TSEGMENTNAME
INTO #bal_prom
FROM rate_calc
WHERE out_rub IS NOT NULL
  AND dt_open BETWEEN @OpenFrom AND @OpenTo;

/* Σ клиента на Anchor (для новых счетов) */
IF OBJECT_ID('tempdb..#cli_sum') IS NOT NULL DROP TABLE #cli_sum;
SELECT cli_id, SUM(out_rub) AS sum_out
INTO   #cli_sum
FROM   #bal_prom
GROUP  BY cli_id;

/* ── 2) Циклы исходных договоров (для ставок и 1-го base-day) ─ */
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

/* ── 3) Дневные ставки для ИСХОДНЫХ договоров (без 1-х чисел) ─ */
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
WHERE  DAY(d.d) <> 1;   -- 1-е числа можно добавить по аналогии, но для 70% не требуется

/* ── 4) ЕЖЕМЕСЯЧНЫЕ base-day события по КЛИЕНТУ до горизонта ──
       (начиная с минимального base_day по его исходным окнам)  */
IF OBJECT_ID('tempdb..#first_base') IS NOT NULL DROP TABLE #first_base;
SELECT cli_id, MIN(base_day) AS first_base_day
INTO   #first_base
FROM   #cycles
GROUP  BY cli_id;

IF OBJECT_ID('tempdb..#events') IS NOT NULL DROP TABLE #events;
;WITH gen AS (
    SELECT cli_id, base_day = first_base_day
    FROM   #first_base
    UNION ALL
    SELECT g.cli_id, EOMONTH(DATEADD(month,1,g.base_day))
    FROM   gen g
    WHERE  EOMONTH(DATEADD(month,1,g.base_day)) <= @HorizonTo
)
SELECT * INTO #events FROM gen OPTION (MAXRECURSION 0);

/* сколько событий наступило к каждой дате — N(t) */
IF OBJECT_ID('tempdb..#event_cnt') IS NOT NULL DROP TABLE #event_cnt;
SELECT
    c.cli_id, d.d AS dt_rep,
    cnt = COUNT(e.base_day)
INTO   #event_cnt
FROM  (SELECT DISTINCT cli_id FROM #bal_prom) c
CROSS JOIN #cal d
LEFT JOIN  #events e
       ON  e.cli_id = c.cli_id AND e.base_day <= d.d
GROUP BY c.cli_id, d.d;

/* ── 5) Масштабируем исходные договоры: (1-Share)^{N(t)} ───── */
IF OBJECT_ID('tempdb..#daily_orig_scaled') IS NOT NULL DROP TABLE #daily_orig_scaled;
SELECT
    dp.con_id, dp.cli_id, dp.TSEGMENTNAME,
    out_rub = CAST(ROUND(dp.out_rub * POWER(1-@ShareNew, ISNULL(ec.cnt,0)), 2) AS decimal(20,2)),
    dp.dt_rep,
    dp.rate_con
INTO #daily_orig_scaled
FROM #daily_pre dp
LEFT JOIN #event_cnt ec
       ON ec.cli_id=dp.cli_id AND ec.dt_rep=dp.dt_rep;

/* ── 6) НОВЫЕ счета на каждом base-day (70% Σ, промо 2 мес.) ─ */
IF OBJECT_ID('tempdb..#newacc_def') IS NOT NULL DROP TABLE #newacc_def;
;WITH pick_rate AS (  -- промо-ставка на base-day: максимум из сегментов
    SELECT
      e.cli_id, e.base_day,
      promo_rate = (SELECT MAX(v) FROM (VALUES (@Spread_DChbo  + k.KEY_RATE),
                                               (@Spread_Retail + k.KEY_RATE)) t(v))
    FROM #events e
    LEFT JOIN #key k ON k.DT_REP = e.base_day
)
SELECT
    new_con_id   = -1 * ABS(CHECKSUM(p.cli_id, p.base_day)), -- синтетический id
    p.cli_id,
    TSEGMENTNAME = N'Промо NEW',
    amount       = CAST(ROUND(cs.sum_out * @ShareNew * POWER(1-@ShareNew,
                       (SELECT COUNT(*) FROM #events e2 WHERE e2.cli_id=p.cli_id AND e2.base_day <  p.base_day)), 2) AS decimal(20,2)),
    win_start    = p.base_day,
    base_day2    = EOMONTH(DATEADD(month,1,p.base_day)),
    promo_end2   = DATEADD(day,-1,EOMONTH(DATEADD(month,1,p.base_day))),
    promo_rate   = p.promo_rate
INTO #newacc_def
FROM pick_rate p
JOIN #cli_sum  cs ON cs.cli_id = p.cli_id;

/* дневка для новых: промо до promo_end2, затем база */
IF OBJECT_ID('tempdb..#newacc_daily') IS NOT NULL DROP TABLE #newacc_daily;
SELECT
    n.new_con_id AS con_id,
    n.cli_id,
    n.TSEGMENTNAME,
    n.amount     AS out_rub,
    d.d          AS dt_rep,
    rate_con     = CASE WHEN d.d <= n.promo_end2 THEN n.promo_rate ELSE @BaseRate END
INTO #newacc_daily
FROM #newacc_def n
JOIN #cal d
  ON d.d BETWEEN n.win_start AND @HorizonTo;

/* ── 7) Итог по дням: исходные (scaled) + новые счета ───────── */
IF OBJECT_ID('WORK.Forecast_NS_Promo','U') IS NOT NULL DROP TABLE WORK.Forecast_NS_Promo;

SELECT
    dt_rep,
    out_rub_total = SUM(out_rub),
    rate_avg      = SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM (
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con FROM #daily_orig_scaled
    UNION ALL
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con FROM #newacc_daily
) u
GROUP BY dt_rep;

/* ── 8) Контрольки ─────────────────────────────────────────── */
PRINT N'=== Σ на Anchor по промо-портфелю (исходные) ===';
SELECT SUM(out_rub) AS sum_anchor FROM #bal_prom;

PRINT N'=== sanity: Σ по дням (должно равняться Σ на Anchor) ===';
SELECT TOP (200) dt_rep, out_rub_total
FROM WORK.Forecast_NS_Promo
ORDER BY dt_rep;
