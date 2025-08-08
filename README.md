/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2 (FIX-promo-roll, prod_id = 654)
   ⚠ СНАПШОТ @Anchor как в mail.usp_fill_balance_metrics_savings:
     rate_use через ULTRA(+1/+2) и LIQ vs balance (те же фильтры).
   v.2025-08-10 — base-day half-shift + full shift on 1st; no dupes
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
SET NOCOUNT ON;

/* ── параметры ─────────────────────────────────────────────── */
DECLARE
    @Scenario  tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;  -- 6.5%

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
  AND  t.prod_id      = 654;                         -- promo-продукт

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
    rate_anchor = CAST(rate_use AS decimal(9,4)),   -- ставка на Anchor из той же логики
    dt_open,
    TSEGMENTNAME
INTO #bal_prom
FROM rate_calc
WHERE out_rub IS NOT NULL
  AND dt_open BETWEEN '2025-07-01' AND '2025-08-31';
-- Если нужно дополнительно исключать «плавающие» договоры, а колонки нет — оставляем как есть.

/* контроль общего объёма на Anchor для sanity */
DECLARE @PromoTotal decimal(20,2) = (SELECT SUM(out_rub) FROM #bal_prom);

/* ── 2) Фикс-спреды по август-25 (из #bal_prom) ─────────────── */
IF OBJECT_ID('WORK.NS_Spreads','U') IS NOT NULL DROP TABLE WORK.NS_Spreads;
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
    @Spread_DChbo   = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО'             THEN w_rate END) - @KeyAtAnchor,
    @Spread_Retail  = MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END) - @KeyAtAnchor;

INSERT WORK.NS_Spreads VALUES
       (N'ДЧБО',            @Spread_DChbo),
       (N'Розничный бизнес',@Spread_Retail);

/* ── 3) Циклы (base_day = EOM(next), promo_end = base_day-1) ── */
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

/* ── 4) Дневная лента без 1-х чисел (promo + base-day) ──────── */
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
                        WHEN N'ДЧБО'             THEN @Spread_DChbo  + k_open.KEY_RATE
                        ELSE                           @Spread_Retail+ k_open.KEY_RATE
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

/* ── 5) BASE-DAY HALF-SHIFT: 50% с «базы» → к счёту с max ставкой ─ */
IF OBJECT_ID('tempdb..#base_dates') IS NOT NULL DROP TABLE #base_dates;
SELECT DISTINCT dp.cli_id, dp.dt_rep
INTO   #base_dates
FROM   #daily_pre dp
JOIN   #cycles c ON c.con_id=dp.con_id AND c.cycle_no=dp.cycle_no
WHERE  dp.dt_rep = c.base_day;

IF OBJECT_ID('tempdb..#daily_base_adj') IS NOT NULL DROP TABLE #daily_base_adj;
;WITH base_pool AS (
    SELECT dp.*,
           is_base = CASE WHEN dp.dt_rep = c.base_day THEN 1 ELSE 0 END
    FROM   #daily_pre dp
    JOIN   #base_dates bd ON bd.cli_id=dp.cli_id AND bd.dt_rep=dp.dt_rep
    LEFT  JOIN #cycles c  ON c.con_id=dp.con_id AND c.cycle_no=dp.cycle_no
),
halves A
