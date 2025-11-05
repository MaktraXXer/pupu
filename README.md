Понял задачу. Делаю «правильно и легко»:
	•	Part 1 → одна компактная таблица интервалов ставок по каждому счёту: WORK.NS_RateIntervals(con_id, cli_id, dt_from, dt_to, rate_con, tsegmentname). Никаких «простыней» по дням и без sys.*
Интервалы строятся из ключевых дат (якорь, все EOM внутри окна, 1-е внутри окна и 1-е нового окна = base_day+1).
	•	Part 2 → только ключевые даты (якорь, EOM, 1-е). Выбор победителя делается строго на этих датах. Между ними значения по определению равны предыдущей ключевой дате, поэтому хранить «междудневку» не нужно.
	•	Для сплита на якоре храню маленькую таблицу WORK.NS_BalPromoAnchor (она крошечная по сравнению с дневной лентой).

Ниже — два монолита. Запускать Part 1 → Part 2 с одинаковыми параметрами.

⸻

PART 1 — интервалы ставок (dt_from/dt_to) по каждому счёту

/* ============================================================
   PART 1: NS (prod_id = 654) — RATE INTERVALS BY ACCOUNT
   Пишет:
     • WORK.NS_BalPromoAnchor(con_id, cli_id, out_rub, tsegmentname)
     • WORK.NS_RateIntervals(con_id, cli_id, tsegmentname, dt_from, dt_to, rate_con)
   Ключевые даты по каждому ЦИКЛУ договора:
     - @Anchor (если попадает в окно цикла)
     - все EOM внутри окна
     - 1-е числа внутри окна
     - 1-е НОВОГО окна (= base_day + 1), если ≤ @HorizonTo
   Между ключевыми датами — постоянная ставка (интервалы).
   ============================================================ */

USE ALM_TEST;
SET NOCOUNT ON;

/* ---------- параметры ---------- */
DECLARE
    @Scenario   tinyint      = 1,
    @Anchor     date         = '2025-11-03',
    @HorizonTo  date         = '2026-03-31',
    @BaseRate   decimal(9,4) = 0.0450,  -- базовая ставка в base_day/EOM
    @UseFixedPromoFromNextMonth bit = 0, -- фикс-промо только в месяце, следующем за якорем
    @UseHardPromoSpread bit         = 0; -- жёсткие спреды вместо расчётных

/* окно фикс-промо: ровно месяц после якоря */
DECLARE @FixedPromoStart date = DATEADD(day,1,EOMONTH(@Anchor));   -- 1-е след. месяца
DECLARE @FixedPromoEnd   date = DATEADD(month,1,@FixedPromoStart); -- 1-е через месяц
DECLARE @HardSpreadStart date = @Anchor;

/* ---------- ключевая ставка ЦБ (TERM=1) ---------- */
IF OBJECT_ID('tempdb..#key') IS NOT NULL DROP TABLE #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@Scenario AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

DECLARE
    @PromoRate_DChbo  decimal(9,4) = 0.1600,
    @PromoRate_Retail decimal(9,4) = 0.1590;

DECLARE @KeyAtAnchor decimal(9,4);
SELECT @KeyAtAnchor = KEY_RATE FROM #key WHERE DT_REP=@Anchor;

DECLARE
    @Spread_DChbo  decimal(9,4) = @PromoRate_DChbo  - @KeyAtAnchor,
    @Spread_Retail decimal(9,4) = @PromoRate_Retail - @KeyAtAnchor;

DECLARE
    @HardSpread_DChbo  decimal(9,4) = -0.0030,
    @HardSpread_Retail decimal(9,4) = -0.0040;

/* ---------- «живые» договоры к якорю ---------- */
DECLARE
    @OpenFrom date = DATEFROMPARTS(YEAR(DATEADD(month,-1,@Anchor)), MONTH(DATEADD(month,-1,@Anchor)), 1),
    @OpenTo   date = EOMONTH(@Anchor);

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
FROM   alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = t.con_id
       AND CASE WHEN t.dt_open=t.dt_rep THEN DATEADD(day,1,t.dt_rep) ELSE t.dt_rep END
           BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day,2,@Anchor)
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.cur          = '810'
  AND  t.prod_id      = 654;

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src(con_id, dt_rep);

/* ---------- ставка на якоре по каждому договору ---------- */
IF OBJECT_ID('tempdb..#bal_prom') IS NOT NULL DROP TABLE #bal_prom;
;WITH bal_pos AS (
    SELECT *,
           MIN(CASE WHEN rate_balance>0 AND rate_con_src=N'счет ультра,вручную'
                    THEN rate_balance END)
           OVER (PARTITION BY con_id
                 ORDER BY dt_rep
                 ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
    FROM #bal_src
    WHERE dt_rep=@Anchor
),
rate_calc AS (
    SELECT *,
      CASE
        WHEN rate_liq IS NULL THEN
             CASE WHEN rate_balance<0 THEN COALESCE(rate_pos, rate_balance)
                  ELSE rate_balance END
        WHEN rate_liq <0 AND rate_balance>0 THEN rate_balance
        WHEN rate_liq <0 AND rate_balance<0 THEN COALESCE(rate_pos, rate_balance)
        WHEN rate_liq>=0 AND rate_balance>=0 THEN rate_liq
        WHEN rate_liq> 0 AND rate_balance<0 THEN rate_liq
        ELSE rate_liq
      END AS rate_use
    FROM bal_pos
)
SELECT
  con_id, cli_id, out_rub,
  rate_anchor = CAST(rate_use AS decimal(9,4)),
  dt_open,
  TSEGMENTNAME
INTO #bal_prom
FROM rate_calc
WHERE out_rub IS NOT NULL
  AND dt_open BETWEEN @OpenFrom AND @OpenTo;

CREATE UNIQUE CLUSTERED INDEX IX_bal_prom ON #bal_prom(con_id);

/* ---------- нулевые договоры к якорю (0 объём, промо ставка по сегменту) ---------- */
IF OBJECT_ID('tempdb..#z0') IS NOT NULL DROP TABLE #z0;
SELECT
  CAST(z.con_id AS bigint) AS con_id,
  CAST(z.cli_id AS bigint) AS cli_id,
  CAST(0.00     AS decimal(20,2)) AS out_rub,
  CAST(COALESCE(z.rate_con,
       CASE WHEN LTRIM(RTRIM(COALESCE(z.TSEGMENTNAME,N'')))=N'ДЧБО'
            THEN @PromoRate_DChbo ELSE @PromoRate_Retail END) AS decimal(9,4)) AS rate_anchor,
  CAST(z.dt_open AS date) AS dt_open,
  CASE WHEN LTRIM(RTRIM(COALESCE(z.TSEGMENTNAME,N'')))=N''
       THEN N'Розничный бизнес' ELSE z.TSEGMENTNAME END AS TSEGMENTNAME
INTO #z0
FROM ALM.ehd.import_ZeroContracts z WITH (NOLOCK)
WHERE z.dt_open<=@Anchor
  AND NOT EXISTS(SELECT 1 FROM #bal_prom p WHERE p.con_id=z.con_id);

INSERT #bal_prom(con_id,cli_id,out_rub,rate_anchor,dt_open,TSEGMENTNAME)
SELECT con_id,cli_id,out_rub,rate_anchor,dt_open,TSEGMENTNAME
FROM #z0;

/* ---------- сохраняем компактный сплит якоря ---------- */
IF OBJECT_ID('WORK.NS_BalPromoAnchor','U') IS NOT NULL DROP TABLE WORK.NS_BalPromoAnchor;
SELECT con_id, cli_id, out_rub, TSEGMENTNAME
INTO WORK.NS_BalPromoAnchor
FROM #bal_prom;

/* ---------- строим циклы окна по каждому договору ---------- */
IF OBJECT_ID('tempdb..#cycles') IS NOT NULL DROP TABLE #cycles;
;WITH seed AS (
  SELECT b.con_id, b.cli_id, b.TSEGMENTNAME, b.out_rub,
         CAST(0 AS int) AS cycle_no,
         b.dt_open AS win_start,
         EOMONTH(DATEADD(month,1,b.dt_open))                  AS base_day,
         DATEADD(day,-1,EOMONTH(DATEADD(month,1,b.dt_open)))  AS promo_end
  FROM #bal_prom b
),
seq AS (
  SELECT * FROM seed
  UNION ALL
  SELECT s.con_id, s.cli_id, s.TSEGMENTNAME, s.out_rub,
         s.cycle_no+1,
         DATEADD(day,1,s.base_day),
         EOMONTH(DATEADD(month,1,DATEADD(day,1,s.base_day))),
         DATEADD(day,-1,EOMONTH(DATEADD(month,1,DATEADD(day,1,s.base_day))))
  FROM seq s
  WHERE DATEADD(day,1,s.base_day) <= @HorizonTo
)
SELECT * INTO #cycles FROM seq OPTION (MAXRECURSION 0);

CREATE UNIQUE CLUSTERED INDEX IX_cycles ON #cycles(con_id, cycle_no);

/* ---------- ключевые даты по КАЖДОМУ циклу ---------- */
IF OBJECT_ID('tempdb..#keys') IS NOT NULL DROP TABLE #keys;
;WITH cal AS (
  SELECT d = @Anchor
  UNION ALL
  SELECT DATEADD(day,1,d) FROM cal WHERE DATEADD(day,1,d) <= @HorizonTo
),
eom_in AS (
  SELECT c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
         k = EOMONTH(x.d)
  FROM #cycles c
  JOIN cal x ON x.d BETWEEN c.win_start AND c.base_day
),
d1_in AS (
  SELECT c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
         k = x.d
  FROM #cycles c
  JOIN cal x ON x.d BETWEEN c.win_start AND c.base_day
  WHERE DAY(x.d)=1
),
d1_new AS (
  SELECT c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
         k = DATEADD(day,1,c.base_day)
  FROM #cycles c
  WHERE DATEADD(day,1,c.base_day) <= @HorizonTo
),
anch AS (
  SELECT c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
         k = @Anchor
  FROM #cycles c
  WHERE @Anchor BETWEEN c.win_start AND c.base_day
)
SELECT DISTINCT con_id, cli_id, TSEGMENTNAME, cycle_no, win_start, promo_end, base_day, k AS dt_key
INTO #keys
FROM (
  SELECT * FROM eom_in
  UNION
  SELECT * FROM d1_in
  UNION
  SELECT * FROM d1_new
  UNION
  SELECT * FROM anch
) U;

CREATE INDEX IX_keys_con_dt ON #keys(con_id, dt_key);

/* ---------- ставка на ключевую дату с приоритетами ---------- */
IF OBJECT_ID('tempdb..#rates_key') IS NOT NULL DROP TABLE #rates_key;
;WITH rk AS (
  SELECT
    x.con_id, x.cli_id, x.TSEGMENTNAME, x.cycle_no, x.win_start, x.promo_end, x.base_day, x.dt_key,
    rate_con =
    CASE
      WHEN x.dt_key = x.base_day THEN @BaseRate
      WHEN DAY(x.dt_key) = 1 THEN
           CASE
             WHEN @UseFixedPromoFromNextMonth=1
              AND x.dt_key>=@FixedPromoStart AND x.dt_key<@FixedPromoEnd THEN
                CASE x.TSEGMENTNAME WHEN N'ДЧБО' THEN @PromoRate_DChbo ELSE @PromoRate_Retail END
             WHEN @UseHardPromoSpread=1 AND x.dt_key>=@HardSpreadStart THEN
                CASE WHEN x.cycle_no=0
                      THEN bp.rate_anchor
                      ELSE CASE x.TSEGMENTNAME
                             WHEN N'ДЧБО' THEN @HardSpread_DChbo  + k.KEY_RATE
                             ELSE               @HardSpread_Retail+ k.KEY_RATE
                           END
                END
             ELSE
                CASE WHEN x.cycle_no=0
                      THEN bp.rate_anchor
                      ELSE CASE x.TSEGMENTNAME
                             WHEN N'ДЧБО' THEN @Spread_DChbo  + k.KEY_RATE
                             ELSE               @Spread_Retail+ k.KEY_RATE
                           END
                END
           END
      ELSE
           /* якорь и «прочие» в окне (не 1-е и не EOM) */
           CASE
             WHEN @UseFixedPromoFromNextMonth=1
              AND x.dt_key>=@FixedPromoStart AND x.dt_key<@FixedPromoEnd THEN
                CASE x.TSEGMENTNAME WHEN N'ДЧБО' THEN @PromoRate_DChbo ELSE @PromoRate_Retail END
             WHEN @UseHardPromoSpread=1 AND x.dt_key>=@HardSpreadStart THEN
                CASE WHEN x.cycle_no=0
                      THEN bp.rate_anchor
                      ELSE CASE x.TSEGMENTNAME
                             WHEN N'ДЧБО' THEN @HardSpread_DChbo  + k.KEY_RATE
                             ELSE               @HardSpread_Retail+ k.KEY_RATE
                           END
                END
             ELSE
                CASE WHEN x.cycle_no=0
                      THEN bp.rate_anchor
                      ELSE CASE x.TSEGMENTNAME
                             WHEN N'ДЧБО' THEN @Spread_DChbo  + k.KEY_RATE
                             ELSE               @Spread_Retail+ k.KEY_RATE
                           END
                END
           END
    END
  FROM #keys x
  LEFT JOIN #key k       ON k.DT_REP = x.dt_key
  JOIN      #bal_prom bp ON bp.con_id = x.con_id
)
SELECT
  con_id, cli_id, TSEGMENTNAME, cycle_no, win_start, promo_end, base_day,
  dt_key, CAST(rate_con AS decimal(9,4)) AS rate_con
INTO #rates_key
FROM rk;

CREATE INDEX IX_rk_con_dt ON #rates_key(con_id, dt_key);

/* ---------- формирование ИНТЕРВАЛОВ ставок ---------- */
IF OBJECT_ID('WORK.NS_RateIntervals','U') IS NOT NULL DROP TABLE WORK.NS_RateIntervals;

;WITH ordered AS (
  SELECT
    con_id, cli_id, TSEGMENTNAME,
    dt_from = dt_key,
    rate_con,
    dt_to_next =
      LEAD(dt_key) OVER (PARTITION BY con_id ORDER BY dt_key)
  FROM #rates_key
),
bounds AS (
  SELECT
    con_id, cli_id, TSEGMENTNAME, rate_con,
    dt_from,
    dt_to = DATEADD(day,-1,ISNULL(dt_to_next, DATEADD(day,1,@HorizonTo))) -- последний тянем до HorizonTo
  FROM ordered
)
SELECT
  con_id, cli_id, TSEGMENTNAME,
  dt_from,
  dt_to,
  rate_con
INTO WORK.NS_RateIntervals
FROM bounds
WHERE dt_from <= @HorizonTo
  AND dt_to   >= @Anchor;

CREATE UNIQUE CLUSTERED INDEX IX_NS_RateIntervals ON WORK.NS_RateIntervals(con_id, dt_from);
CREATE INDEX IX_NS_RateIntervals_cli ON WORK.NS_RateIntervals(cli_id, dt_from);


⸻

PART 2 — аллокации только на ключевых датах (якорь, EOM, 1-е)

/* ================================================================
   PART 2: NS ALLOC ON KEY DATES (ANCHOR + EOM + 1ST)
   Источники:
     • WORK.NS_BalPromoAnchor(con_id, cli_id, out_rub, tsegmentname)
     • WORK.NS_RateIntervals(con_id, cli_id, tsegmentname, dt_from, dt_to, rate_con)
   Выход:
     • WORK.NS_AllocEvents(cli_id, dt_rep, con_id, out_rub, reason)
     • WORK.Forecast_NS_Promo (ТОЛЬКО на ключевые даты: anchor, EOM, 1st)
   Между ключевыми датами значения равны предыдущей ключевой дате
   (поэтому отдельно не храним).
   ================================================================ */

USE ALM_TEST;
SET NOCOUNT ON;

/* ---------- параметры (должны совпадать с Part 1) ---------- */
DECLARE
    @Anchor     date = '2025-11-03',
    @HorizonTo  date = '2026-03-31';

/* ---------- проверки ---------- */
IF OBJECT_ID('WORK.NS_RateIntervals','U') IS NULL
  RAISERROR('Missing WORK.NS_RateIntervals from Part 1.',16,1);

IF OBJECT_ID('WORK.NS_BalPromoAnchor','U') IS NULL
  RAISERROR('Missing WORK.NS_BalPromoAnchor from Part 1.',16,1);

/* ---------- компактные копии ---------- */
IF OBJECT_ID('tempdb..#ri') IS NOT NULL DROP TABLE #ri;
SELECT
  con_id   = CAST(con_id AS bigint),
  cli_id   = CAST(cli_id AS bigint),
  dt_from  = CAST(dt_from AS date),
  dt_to    = CAST(dt_to   AS date),
  rate_con = CAST(rate_con AS decimal(9,4))
INTO #ri
FROM WORK.NS_RateIntervals;

CREATE INDEX IX_ri_con_span ON #ri(con_id, dt_from, dt_to);
CREATE INDEX IX_ri_cli_span ON #ri(cli_id, dt_from, dt_to);

IF OBJECT_ID('tempdb..#ab') IS NOT NULL DROP TABLE #ab;
SELECT
  con_id  = CAST(con_id AS bigint),
  cli_id  = CAST(cli_id AS bigint),
  out_rub = CAST(out_rub AS decimal(20,2))
INTO #ab
FROM WORK.NS_BalPromoAnchor;

CREATE INDEX IX_ab_cli ON #ab(cli_id, con_id);

/* ---------- ключевые даты (anchor + все EOM + все 1-е) ---------- */
IF OBJECT_ID('tempdb..#keydates') IS NOT NULL DROP TABLE #keydates;
;WITH cal AS (
  SELECT d = @Anchor
  UNION ALL
  SELECT DATEADD(day,1,d) FROM cal WHERE DATEADD(day,1,d) <= @HorizonTo
),
eom AS (
  SELECT d = EOMONTH(d) FROM cal
),
d1 AS (
  SELECT d FROM cal WHERE DAY(d)=1
)
SELECT DISTINCT d
INTO #keydates
FROM (
  SELECT @Anchor AS d
  UNION
  SELECT d FROM eom
  UNION
  SELECT d FROM d1
) u;

CREATE UNIQUE INDEX IX_keydates ON #keydates(d);

/* ---------- суммы клиента на якоре ---------- */
IF OBJECT_ID('tempdb..#cli_sum') IS NOT NULL DROP TABLE #cli_sum;
SELECT cli_id, out_rub_sum = SUM(out_rub)
INTO #cli_sum
FROM #ab
GROUP BY cli_id;

CREATE UNIQUE INDEX IX_clisum ON #cli_sum(cli_id);

/* ---------- кандидаты-счета на каждую ключевую дату ---------- */
IF OBJECT_ID('tempdb..#cand') IS NOT NULL DROP TABLE #cand;
SELECT
  kd.d       AS dt_rep,
  ab.cli_id,
  ab.con_id
INTO #cand
FROM #keydates kd
CROSS JOIN (SELECT DISTINCT cli_id FROM #ab) C
JOIN #ab ab ON ab.cli_id = C.cli_id;

CREATE INDEX IX_cand_cli_dt ON #cand(cli_id, dt_rep);
CREATE INDEX IX_cand_con_dt ON #cand(con_id, dt_rep);

/* ---------- ставка счёта на ключевую дату из ИНТЕРВАЛОВ ---------- */
IF OBJECT_ID('tempdb..#cand_rates') IS NOT NULL DROP TABLE #cand_rates;
SELECT
  c.dt_rep,
  c.cli_id,
  c.con_id,
  rate_on_date = ri.rate_con
INTO #cand_rates
FROM #cand c
JOIN #ri ri
  ON ri.con_id = c.con_id
 AND c.dt_rep BETWEEN ri.dt_from AND ri.dt_to;

CREATE INDEX IX_cr_cli_dt ON #cand_rates(cli_id, dt_rep, rate_on_date DESC, con_id);

/* ---------- события перелива: EOM и 1-е ---------- */
IF OBJECT_ID('tempdb..#evt') IS NOT NULL DROP TABLE #evt;
;WITH eom AS (
  SELECT DISTINCT dt_rep FROM #keydates WHERE dt_rep = EOMONTH(dt_rep)
),
d1 AS (
  SELECT DISTINCT dt_rep FROM #keydates WHERE DAY(dt_rep)=1
),
ranked_base AS (
  SELECT
    r.cli_id, r.dt_rep, r.con_id, r.rate_on_date,
    rn = ROW_NUMBER() OVER(PARTITION BY r.cli_id, r.dt_rep
                           ORDER BY r.rate_on_date DESC, r.con_id)
  FROM #cand_rates r
  WHERE r.dt_rep IN (SELECT dt_rep FROM eom)
),
ranked_first AS (
  SELECT
    r.cli_id, r.dt_rep, r.con_id, r.rate_on_date,
    rn = ROW_NUMBER() OVER(PARTITION BY r.cli_id, r.dt_rep
                           ORDER BY r.rate_on_date DESC, r.con_id)
  FROM #cand_rates r
  WHERE r.dt_rep IN (SELECT dt_rep FROM d1)
)
SELECT
  cli_id,
  con_id,
  dt_rep,
  out_rub = cs.out_rub_sum,
  reason  = CAST('EOM' AS varchar(8))
INTO #evt
FROM ranked_base rb
JOIN #cli_sum cs ON cs.cli_id = rb.cli_id
WHERE rb.rn = 1

UNION ALL

SELECT
  cli_id,
  con_id,
  dt_rep,
  out_rub = cs.out_rub_sum,
  reason  = CAST('D1' AS varchar(8))
FROM ranked_first rf
JOIN #cli_sum cs ON cs.cli_id = rf.cli_id
WHERE rf.rn = 1;

CREATE INDEX IX_evt_cli_dt ON #evt(cli_id, dt_rep);

/* ---------- materialize events ---------- */
IF OBJECT_ID('WORK.NS_AllocEvents','U') IS NOT NULL DROP TABLE WORK.NS_AllocEvents;
SELECT cli_id, dt_rep, con_id, out_rub, reason
INTO WORK.NS_AllocEvents
FROM #evt;

CREATE INDEX IX_aevt_cli_dt ON WORK.NS_AllocEvents(cli_id, dt_rep);
CREATE INDEX IX_aevt_con_dt ON WORK.NS_AllocEvents(con_id, dt_rep);

/* ---------- аллокация ТОЛЬКО на ключевых датах -------------- */
/* До первой ключевой даты клиента — якорный сплит (только если anchor ∈ keydates).
   Начиная с первой его ключевой даты — весь Σ клиента на победителе последнего события. */

IF OBJECT_ID('tempdb..#first_evt') IS NOT NULL DROP TABLE #first_evt;
SELECT cli_id, first_evt = MIN(dt_rep)
INTO #first_evt
FROM WORK.NS_AllocEvents
GROUP BY cli_id;

CREATE UNIQUE INDEX IX_fe_cli ON #first_evt(cli_id);

/* Якорь как ключевая дата */
DECLARE @HasAnchorKey bit = CASE WHEN EXISTS(SELECT 1 FROM #keydates WHERE d=@Anchor) THEN 1 ELSE 0 END;

/* Аллокация на ключевых датах */
IF OBJECT_ID('tempdb..#alloc_key') IS NOT NULL DROP TABLE #alloc_key;
-- 1) если якорь есть в keydates и у клиента ещё не было событий — держим якорный сплит на якоре
SELECT
  kd.d       AS dt_rep,
  ab.cli_id,
  ab.con_id,
  ab.out_rub
INTO #alloc_key
FROM #keydates kd
JOIN #ab ab ON @HasAnchorKey=1 AND kd.d=@Anchor
LEFT JOIN #first_evt fe ON fe.cli_id = ab.cli_id
WHERE fe.cli_id IS NULL OR fe.first_evt > @Anchor

UNION ALL
-- 2) на любой ключевой дате >= первой даты события берём ПОСЛЕДНЕГО победителя
SELECT
  kd.d       AS dt_rep,
  cl.cli_id,
  win.con_id,
  out_rub = cs.out_rub_sum
FROM #keydates kd
JOIN (SELECT DISTINCT cli_id FROM #ab) cl ON 1=1
JOIN #cli_sum cs ON cs.cli_id = cl.cli_id
LEFT JOIN #first_evt fe
  ON fe.cli_id = cl.cli_id
OUTER APPLY (
  SELECT TOP (1) e.con_id
  FROM WORK.NS_AllocEvents e
  WHERE e.cli_id = cl.cli_id
    AND e.dt_rep <= kd.d
  ORDER BY e.dt_rep DESC, e.con_id
) win
WHERE win.con_id IS NOT NULL   -- есть уже «последний победитель»
  AND kd.d >= fe.first_evt;    -- после появления первой даты события

CREATE INDEX IX_alloc_key_dt ON #alloc_key(dt_rep);
CREATE INDEX IX_alloc_key_con_dt ON #alloc_key(con_id, dt_rep);

/* ---------- ставка на ключевую дату (по интервалам) ---------- */
IF OBJECT_ID('tempdb..#rates_keydates') IS NOT NULL DROP TABLE #rates_keydates;
SELECT
  a.dt_rep,
  a.con_id,
  rate_con = ri.rate_con
INTO #rates_keydates
FROM #alloc_key a
JOIN #ri ri
  ON ri.con_id = a.con_id
 AND a.dt_rep BETWEEN ri.dt_from AND ri.dt_to;

CREATE INDEX IX_rkd_con_dt ON #rates_keydates(con_id, dt_rep);

/* ---------- итог: агрегат ТОЛЬКО на ключевые даты ---------- */
IF OBJECT_ID('WORK.Forecast_NS_Promo','U') IS NOT NULL DROP TABLE WORK.Forecast_NS_Promo;

SELECT
  a.dt_rep,
  out_rub_total = SUM(a.out_rub),
  rate_avg      = SUM(a.out_rub * CAST(r.rate_con AS decimal(9,4))) / NULLIF(SUM(a.out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM #alloc_key a
JOIN #rates_keydates r
  ON r.con_id = a.con_id
 AND r.dt_rep = a.dt_rep
GROUP BY a.dt_rep;

/* ---------- контрольки (коротко) ---------- */
PRINT N'=== Events (top-50) ===';
SELECT TOP (50) * FROM WORK.NS_AllocEvents ORDER BY cli_id, dt_rep;

PRINT N'=== Forecast (key dates only, top-200) ===';
SELECT TOP (200) * FROM WORK.Forecast_NS_Promo ORDER BY dt_rep;

DECLARE @AnchorTotal decimal(20,2) = (SELECT SUM(out_rub) FROM #ab);
PRINT N'=== Σ(out_rub) vs Anchor on key dates ===';
SELECT TOP (200)
  f.dt_rep,
  f.out_rub_total,
  diff_vs_anchor = CAST(f.out_rub_total - @AnchorTotal AS decimal(20,2)),
  f.rate_avg
FROM WORK.Forecast_NS_Promo f
ORDER BY f.dt_rep;


⸻

Почему это оптимально
	•	Part 1 хранит только интервалы (обычно считанные записи на счёт за горизонт), а не посуточку.
	•	Part 2 считает только на ключевые даты (anchor/EOM/1st). Между ними всё по определению «ступенька» от последней ключевой точки — можно восстановить «на лету» при необходимости (вью/запросом), не кладя в таблицу.
	•	Все джоины — по компактным наборам с узкими индексами; верхний bound по датам между dt_from/dt_to даёт гарантированно индексируемый seek.
	•	Никаких sys.*.

Если захочешь — добавлю лёгкую вьюшку vw_Forecast_NS_Promo_Daily, которая будет «размазывать» ключевые значения на каждый день только на чтение, без хранения.
