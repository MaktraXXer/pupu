Ты прав: части не стыковались по двум причинам:
	1.	Part 1 не сохранял снапшот якорного сплита в WORK.NS_BalPromoAnchor, а Part 2 его требует.
	2.	В Part 1 на каждый цикл не попадал «1-е нового окна» (base_day + 1) — из-за условия k.d BETWEEN win_start AND base_day. Из-за этого Part 2 видел не все 1-е числа.

Ниже даю два законченных скрипта (Part 1 → Part 2), которые дружат между собой:
	•	Part 1 делает: WORK.NS_BalPromoAnchor и WORK.NS_RatesKeyDates, причём на каждый цикл кладёт ключевые даты: @Anchor (если внутри окна), все EOM внутри окна, 1-е внутри окна и 1-е нового окна = base_day+1 (если ≤ @HorizonTo).
	•	Part 2 берёт только EOM и все 1-е из NS_RatesKeyDates, считает победителя и формирует дневную ленту (ступеньками). В ранжировании исключены NULL-ставки.

⸻

PART 1 — RATES ONLY (память ставок на ключевые даты)

/* ============================================================
   PART 1: NS (prod_id = 654) — RATES ONLY (key-dates memory)
   Пишет:
     • WORK.NS_BalPromoAnchor(con_id, cli_id, out_rub, TSEGMENTNAME)
     • WORK.NS_RatesKeyDates(con_id, cli_id, dt_key, rate_con, …)
   Ключевые даты по КАЖДОМУ циклу договора:
     - @Anchor (если попадает в окно цикла)
     - все EOM внутри окна
     - 1-е внутри окна
     - 1-е НОВОГО окна (= base_day + 1), если ≤ @HorizonTo
   ============================================================ */

USE ALM_TEST;
SET NOCOUNT ON;

/* ---------- параметры ---------- */
DECLARE
    @Scenario   tinyint      = 1,
    @Anchor     date         = '2025-11-03',
    @HorizonTo  date         = '2026-03-31',
    @BaseRate   decimal(9,4) = 0.0450,
    @UseFixedPromoFromNextMonth bit = 0,
    @UseHardPromoSpread bit         = 0;

/* окно фикс-промо: ровно месяц после якоря */
DECLARE @FixedPromoStart date = DATEADD(day,1,EOMONTH(@Anchor));   -- 1-е след. месяца
DECLARE @FixedPromoEnd   date = DATEADD(month,1,@FixedPromoStart); -- 1-е через месяц
/* старт жёстких спредов */
DECLARE @HardSpreadStart date = @Anchor;

/* ---------- календарь всего диапазона ---------- */
IF OBJECT_ID('tempdb..#cal_all') IS NOT NULL DROP TABLE #cal_all;
;WITH d AS (
  SELECT d = @Anchor
  UNION ALL SELECT DATEADD(day,1,d) FROM d WHERE DATEADD(day,1,d) <= @HorizonTo
)
SELECT d INTO #cal_all FROM d OPTION (MAXRECURSION 0);

/* ---------- ключевая ставка (TERM=1) ---------- */
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

/* ---------- границы «живых» договоров ---------- */
DECLARE
    @OpenFrom date = DATEFROMPARTS(YEAR(DATEADD(month,-1,@Anchor)), MONTH(DATEADD(month,-1,@Anchor)), 1),
    @OpenTo   date = EOMONTH(@Anchor);

/* ---------- снапшот промо-портфеля на якоре ---------- */
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

CREATE UNIQUE CLUSTERED INDEX IX_bal_prom ON #bal_prom(con_id);

/* ---------- ПЕРСИСТИМ сплит якоря для Part 2 ---------- */
IF OBJECT_ID('WORK.NS_BalPromoAnchor','U') IS NOT NULL DROP TABLE WORK.NS_BalPromoAnchor;
SELECT con_id, cli_id, out_rub, TSEGMENTNAME
INTO WORK.NS_BalPromoAnchor
FROM #bal_prom;

/* ---------- построение циклов ---------- */
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
/* Для каждого цикла формируем:
     • @Anchor (если @Anchor BETWEEN win_start AND base_day)
     • все EOM внутри [win_start .. base_day]
     • 1-е внутри [win_start .. base_day]
     • 1-е НОВОГО окна = base_day + 1 (если ≤ @HorizonTo)         */

IF OBJECT_ID('tempdb..#keys_this_cycle') IS NOT NULL DROP TABLE #keys_this_cycle;
;WITH eom_in AS (
  SELECT c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
         k = EOMONTH(x.d)
  FROM #cycles c
  JOIN #cal_all x ON x.d BETWEEN c.win_start AND c.base_day
),
d1_in AS (
  SELECT c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
         k = x.d
  FROM #cycles c
  JOIN #cal_all x ON x.d BETWEEN c.win_start AND c.base_day
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
INTO #keys_this_cycle
FROM (
  SELECT * FROM eom_in
  UNION
  SELECT * FROM d1_in
  UNION
  SELECT * FROM d1_new
  UNION
  SELECT * FROM anch
) U;

CREATE INDEX IX_keysc_con_dt ON #keys_this_cycle(con_id, dt_key);

/* ---------- расчёт ставки на dt_key с приоритетом ---------- */
IF OBJECT_ID('WORK.NS_RatesKeyDates','U') IS NOT NULL DROP TABLE WORK.NS_RatesKeyDates;

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
           /* якорь и прочие внутриокновые (не 1-е и не EOM) */
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
  FROM #keys_this_cycle x
  LEFT JOIN #key k       ON k.DT_REP = x.dt_key
  JOIN      #bal_prom bp ON bp.con_id = x.con_id
)
SELECT
  con_id, cli_id, TSEGMENTNAME, cycle_no, win_start, promo_end, base_day,
  dt_key, CAST(rate_con AS decimal(9,4)) AS rate_con
INTO WORK.NS_RatesKeyDates
FROM rk;

CREATE UNIQUE CLUSTERED INDEX IX_NS_RatesKeyDates ON WORK.NS_RatesKeyDates(con_id, dt_key);
CREATE INDEX IX_NS_RatesKeyDates_cli ON WORK.NS_RatesKeyDates(cli_id, dt_key);


⸻

PART 2 — ALLOC (перелив объёмов по ключевым датам)

/* ================================================================
   NS-forecast — Part 2 (ALLOC)
   Зависимости:
     • WORK.NS_RatesKeyDates(con_id, cli_id, dt_key, rate_con, …)
     • WORK.NS_BalPromoAnchor(con_id, cli_id, out_rub, TSEGMENTNAME)
   Логика:
     • События только на EOM и всех 1-х (включая base_day+1).
     • До первого события — держим якорный сплит.
     • После — весь Σ клиента на последнем победителе.
     • Ставка счёта на день — последняя ключевая ≤ дата.
   ================================================================ */

SET NOCOUNT ON;

/* ------------------ sanity deps ------------------ */
IF OBJECT_ID('WORK.NS_RatesKeyDates','U') IS NULL
  RAISERROR('Missing WORK.NS_RatesKeyDates from Part 1.',16,1);

IF OBJECT_ID('WORK.NS_BalPromoAnchor','U') IS NULL
  RAISERROR('Missing WORK.NS_BalPromoAnchor from Part 1.',16,1);

/* ------------------ source copies ---------------- */
IF OBJECT_ID('tempdb..#rates_key') IS NOT NULL DROP TABLE #rates_key;
SELECT
  con_id   = CAST(con_id AS bigint),
  cli_id   = CAST(cli_id AS bigint),
  dt_key   = CAST(dt_key AS date),
  rate_con = CAST(rate_con AS decimal(9,4))
INTO #rates_key
FROM WORK.NS_RatesKeyDates;

CREATE INDEX IX_rk_con_dt ON #rates_key(con_id, dt_key);
CREATE INDEX IX_rk_cli_dt ON #rates_key(cli_id, dt_key);

IF OBJECT_ID('tempdb..#anchor_bal') IS NOT NULL DROP TABLE #anchor_bal;
SELECT
  con_id  = CAST(con_id AS bigint),
  cli_id  = CAST(cli_id AS bigint),
  out_rub = CAST(out_rub AS decimal(20,2))
INTO #anchor_bal
FROM WORK.NS_BalPromoAnchor;

CREATE INDEX IX_ab_cli_con ON #anchor_bal(cli_id, con_id);

/* диапазон дат из ключей */
DECLARE @StartDate date, @EndDate date;
SELECT @StartDate = MIN(dt_key), @EndDate = MAX(dt_key) FROM #rates_key;

IF @StartDate IS NULL OR @EndDate IS NULL
  RAISERROR('NS_RatesKeyDates contains no rows.',16,1);

/* ------------ набор клиентов и суммы на якоре ------------- */
IF OBJECT_ID('tempdb..#clients') IS NOT NULL DROP TABLE #clients;
SELECT DISTINCT cli_id INTO #clients FROM #anchor_bal;
CREATE UNIQUE INDEX IX_cli ON #clients(cli_id);

IF OBJECT_ID('tempdb..#cli_sum') IS NOT NULL DROP TABLE #cli_sum;
SELECT cli_id, out_rub_sum = SUM(out_rub)
INTO #cli_sum
FROM #anchor_bal
GROUP BY cli_id;
CREATE UNIQUE INDEX IX_clisum ON #cli_sum(cli_id);

/* -------- ключевые даты: EOM и 1-е (строго из NS_RatesKeyDates) ----- */
IF OBJECT_ID('tempdb..#d_eom') IS NOT NULL DROP TABLE #d_eom;
SELECT DISTINCT d = rk.dt_key
INTO #d_eom
FROM #rates_key rk
WHERE rk.dt_key = EOMONTH(rk.dt_key);

IF OBJECT_ID('tempdb..#d_first') IS NOT NULL DROP TABLE #d_first;
SELECT DISTINCT d = rk.dt_key
INTO #d_first
FROM #rates_key rk
WHERE DAY(rk.dt_key) = 1;

/* --------- календарь (ступени) -------- */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
;WITH span AS (
  SELECT d = @StartDate
  UNION ALL
  SELECT DATEADD(day,1,d) FROM span WHERE DATEADD(day,1,d) <= @EndDate
)
SELECT d INTO #cal FROM span OPTION (MAXRECURSION 0);

CREATE UNIQUE INDEX IX_cal ON #cal(d);

/* ------- кандидаты-счета на ключевые даты (по клиентам) ------ */
IF OBJECT_ID('tempdb..#acc_candidates') IS NOT NULL DROP TABLE #acc_candidates;
SELECT
  c.cli_id,
  ab.con_id,
  kd.d AS dt_rep
INTO #acc_candidates
FROM #clients c
JOIN #anchor_bal ab ON ab.cli_id = c.cli_id
JOIN (
  SELECT d FROM #d_eom
  UNION
  SELECT d FROM #d_first
) kd ON 1=1;

CREATE INDEX IX_cand_cli_dt ON #acc_candidates(cli_id, dt_rep, con_id);

/* ставка счёта на ключевую дату = последняя ключевая ≤ дата */
IF OBJECT_ID('tempdb..#acc_rates_on_key') IS NOT NULL DROP TABLE #acc_rates_on_key;
SELECT
  ac.cli_id,
  ac.con_id,
  ac.dt_rep,
  rate_on_date = oa.rate_con
INTO #acc_rates_on_key
FROM #acc_candidates ac
OUTER APPLY (
  SELECT TOP (1) rk.rate_con
  FROM #rates_key rk
  WHERE rk.con_id = ac.con_id
    AND rk.dt_key <= ac.dt_rep
  ORDER BY rk.dt_key DESC
) oa;

CREATE INDEX IX_arok_cli_dt ON #acc_rates_on_key(cli_id, dt_rep, rate_on_date DESC, con_id);

/* ------------------- события: EOM ------------------- */
IF OBJECT_ID('tempdb..#evt_base') IS NOT NULL DROP TABLE #evt_base;
;WITH ranked AS (
  SELECT
    a.cli_id, a.dt_rep, a.con_id, a.rate_on_date,
    rn = ROW_NUMBER() OVER (
           PARTITION BY a.cli_id, a.dt_rep
           ORDER BY a.rate_on_date DESC, a.con_id
         )
  FROM #acc_rates_on_key a
  WHERE a.dt_rep IN (SELECT d FROM #d_eom)
    AND a.rate_on_date IS NOT NULL
)
SELECT
  r.cli_id,
  r.con_id,
  r.dt_rep,
  out_rub = cs.out_rub_sum,
  reason  = CAST('EOM' AS varchar(8))
INTO #evt_base
FROM ranked r
JOIN #cli_sum cs ON cs.cli_id = r.cli_id
WHERE r.rn = 1;

CREATE INDEX IX_evtb_cli_dt ON #evt_base(cli_id, dt_rep);

/* ------------------- события: 1-е ------------------- */
IF OBJECT_ID('tempdb..#evt_first') IS NOT NULL DROP TABLE #evt_first;
;WITH ranked AS (
  SELECT
    a.cli_id, a.dt_rep, a.con_id, a.rate_on_date,
    rn = ROW_NUMBER() OVER (
           PARTITION BY a.cli_id, a.dt_rep
           ORDER BY a.rate_on_date DESC, a.con_id
         )
  FROM #acc_rates_on_key a
  WHERE a.dt_rep IN (SELECT d FROM #d_first)
    AND a.rate_on_date IS NOT NULL
)
SELECT
  r.cli_id,
  r.con_id,
  r.dt_rep,
  out_rub = cs.out_rub_sum,
  reason  = CAST('D1' AS varchar(8))
INTO #evt_first
FROM ranked r
JOIN #cli_sum cs ON cs.cli_id = r.cli_id
WHERE r.rn = 1;

CREATE INDEX IX_evtf_cli_dt ON #evt_first(cli_id, dt_rep);

/* --------- итоговый пул событий клиента (материализация) ------ */
IF OBJECT_ID('WORK.NS_AllocEvents','U') IS NOT NULL DROP TABLE WORK.NS_AllocEvents;

SELECT cli_id, con_id, dt_rep, out_rub, reason
INTO WORK.NS_AllocEvents
FROM (
  SELECT * FROM #evt_base
  UNION ALL
  SELECT * FROM #evt_first
) u;

CREATE INDEX IX_aevt_cli_dt ON WORK.NS_AllocEvents(cli_id, dt_rep);
CREATE INDEX IX_aevt_con_dt ON WORK.NS_AllocEvents(con_id, dt_rep);

/* --------- «первая дата события» по каждому клиенту ---------- */
IF OBJECT_ID('tempdb..#first_evt') IS NOT NULL DROP TABLE #first_evt;
SELECT cli_id, first_evt = MIN(dt_rep)
INTO #first_evt
FROM WORK.NS_AllocEvents
GROUP BY cli_id;

CREATE UNIQUE INDEX IX_fe_cli ON #first_evt(cli_id);

/* -------------- решётка (день × клиент) ---------------------- */
IF OBJECT_ID('tempdb..#day_cli') IS NOT NULL DROP TABLE #day_cli;
SELECT c.d AS dt_rep, cl.cli_id
INTO #day_cli
FROM #cal c
CROSS JOIN #clients cl;

CREATE INDEX IX_daycli_cli_dt ON #day_cli(cli_id, dt_rep);

/* ----------- аллокации до первого события: сплит якоря ------ */
IF OBJECT_ID('tempdb..#alloc_pre') IS NOT NULL DROP TABLE #alloc_pre;
-- клиенты с событиями: до first_evt держим якорный сплит
SELECT
  dc.dt_rep,
  ab.cli_id,
  ab.con_id,
  ab.out_rub
INTO #alloc_pre
FROM #day_cli dc
JOIN #first_evt fe ON fe.cli_id = dc.cli_id
JOIN #anchor_bal ab ON ab.cli_id = dc.cli_id
WHERE dc.dt_rep < fe.first_evt

UNION ALL
-- клиенты без событий: весь горизонт держим якорный сплит
SELECT
  dc.dt_rep,
  ab.cli_id,
  ab.con_id,
  ab.out_rub
FROM #day_cli dc
JOIN #anchor_bal ab ON ab.cli_id = dc.cli_id
WHERE NOT EXISTS (SELECT 1 FROM #first_evt fe WHERE fe.cli_id = dc.cli_id);

/* ----------- аллокации после первого события: winner ---------- */
IF OBJECT_ID('tempdb..#alloc_post') IS NOT NULL DROP TABLE #alloc_post;
SELECT
  dc.dt_rep,
  dc.cli_id,
  win.con_id,
  out_rub = cs.out_rub_sum
INTO #alloc_post
FROM #day_cli dc
JOIN #first_evt fe ON fe.cli_id = dc.cli_id
  AND dc.dt_rep >= fe.first_evt
JOIN #cli_sum cs ON cs.cli_id = dc.cli_id
OUTER APPLY (
  SELECT TOP (1) ae.con_id
  FROM WORK.NS_AllocEvents ae
  WHERE ae.cli_id = dc.cli_id
    AND ae.dt_rep <= dc.dt_rep
  ORDER BY ae.dt_rep DESC, ae.con_id
) win;

/* ------------------- общий дневной пул ----------------------- */
IF OBJECT_ID('tempdb..#alloc_daily') IS NOT NULL DROP TABLE #alloc_daily;
SELECT * INTO #alloc_daily FROM #alloc_pre;
INSERT #alloc_daily (dt_rep, cli_id, con_id, out_rub)
SELECT dt_rep, cli_id, con_id, out_rub FROM #alloc_post;

CREATE INDEX IX_ad_dt ON #alloc_daily(dt_rep);
CREATE INDEX IX_ad_con_dt ON #alloc_daily(con_id, dt_rep);

/* ставка счёта на день = последняя ключевая ≤ дата */
IF OBJECT_ID('tempdb..#rates_daily') IS NOT NULL DROP TABLE #rates_daily;
SELECT
  ad.dt_rep,
  ad.con_id,
  rate_con = oa.rate_con
INTO #rates_daily
FROM #alloc_daily ad
OUTER APPLY (
  SELECT TOP (1) rk.rate_con
  FROM #rates_key rk
  WHERE rk.con_id = ad.con_id
    AND rk.dt_key <= ad.dt_rep
  ORDER BY rk.dt_key DESC
) oa;

CREATE INDEX IX_rd_con_dt ON #rates_daily(con_id, dt_rep);

/* ------------------- итоговая агрегация ---------------------- */
IF OBJECT_ID('WORK.Forecast_NS_Promo','U') IS NOT NULL DROP TABLE WORK.Forecast_NS_Promo;

SELECT
  ad.dt_rep,
  out_rub_total = SUM(ad.out_rub),
  rate_avg      = SUM(ad.out_rub * CAST(rd.rate_con AS decimal(9,4))) / NULLIF(SUM(ad.out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM #alloc_daily ad
JOIN #rates_daily rd
  ON rd.con_id = ad.con_id
 AND rd.dt_rep = ad.dt_rep
GROUP BY ad.dt_rep;

Что изменено по сути:
	•	Part 1 теперь всегда даёт «1-е нового окна» (base_day+1) в NS_RatesKeyDates — без этого второй шаг не видит перезапуск промо.
	•	Part 1 сохраняет WORK.NS_BalPromoAnchor.
	•	Part 2 при выборе победителя фильтрует rate_on_date IS NOT NULL, чтобы не было «тихих» пустых рангов.
	•	Нигде нет обращений к sys.*.
	•	Ступенчатая дневная лента: между событиями держится последний победитель; «внутримесячные» дни не пересчитываются.

Запускай строго: Part 1 → Part 2 (с одинаковыми параметрами @Anchor/@HorizonTo). Если понадобится — добавлю лёгкую «контрольку» сравнения Σобъёма и выборку победителей по любому cli_id.
