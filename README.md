/* ============================================================
   PART 2: NS — ALLOCATION EVENTS & DAILY AGGREGATE
   Вход:
     • WORK.NS_BalPromoAnchor
     • WORK.NS_RateIntervals
   Выход:
     • WORK.NS_AllocEvents
     • WORK.Forecast_NS_Promo
   Правила:
     • Победитель выбирается ТОЛЬКО на EOM и 1-е
       (ставка счёта на дату — по интервалам Part 1, OUTER APPLY).
     • До первого события — держим якорный сплит по кон-ам.
     • После события — весь Σ клиента на последнем победителе.
   ============================================================ */

USE ALM_TEST;
SET NOCOUNT ON;

/* ---------- проверки ---------- */
IF OBJECT_ID('WORK.NS_RateIntervals','U') IS NULL
  RAISERROR('Missing WORK.NS_RateIntervals (run Part 1).',16,1);

IF OBJECT_ID('WORK.NS_BalPromoAnchor','U') IS NULL
  RAISERROR('Missing WORK.NS_BalPromoAnchor (run Part 1).',16,1);

/* ---------- диапазон дат ---------- */
DECLARE @StartDate date, @EndDate date;
SELECT @StartDate = MIN(dt_from), @EndDate = MAX(dt_to) FROM WORK.NS_RateIntervals;

/* ---------- календарь только по ключевым (EOM/1-е) + анкоры в интервалах ---------- */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
CREATE TABLE #cal (d date NOT NULL PRIMARY KEY);
INSERT #cal VALUES (@StartDate);
WHILE (SELECT MAX(d) FROM #cal) < @EndDate
BEGIN
  INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;
END;

IF OBJECT_ID('tempdb..#eom') IS NOT NULL DROP TABLE #eom;
SELECT d INTO #eom FROM #cal WHERE d = EOMONTH(d);

IF OBJECT_ID('tempdb..#d1') IS NOT NULL DROP TABLE #d1;
SELECT d INTO #d1 FROM #cal WHERE DAY(d)=1;

/* ---------- клиенты и суммы на якоре ---------- */
IF OBJECT_ID('tempdb..#anchor_bal') IS NOT NULL DROP TABLE #anchor_bal;
SELECT con_id=CAST(con_id AS bigint),
       cli_id=CAST(cli_id AS bigint),
       out_rub=CAST(out_rub AS decimal(20,2))
INTO #anchor_bal
FROM WORK.NS_BalPromoAnchor;

IF OBJECT_ID('tempdb..#clients') IS NOT NULL DROP TABLE #clients;
SELECT DISTINCT cli_id INTO #clients FROM #anchor_bal;
CREATE UNIQUE INDEX IX_cli ON #clients(cli_id);

IF OBJECT_ID('tempdb..#cli_sum') IS NOT NULL DROP TABLE #cli_sum;
SELECT cli_id, out_rub_sum = SUM(out_rub)
INTO #cli_sum
FROM #anchor_bal
GROUP BY cli_id;
CREATE UNIQUE INDEX IX_clisum ON #cli_sum(cli_id);

/* ---------- кандидаты «счет×клиент×ключевая дата» ---------- */
IF OBJECT_ID('tempdb..#candidates') IS NOT NULL DROP TABLE #candidates;
SELECT c.cli_id, ab.con_id, k.d AS dt_rep
INTO #candidates
FROM #clients c
JOIN #anchor_bal ab ON ab.cli_id = c.cli_id
JOIN (SELECT d FROM #eom UNION ALL SELECT d FROM #d1) k ON 1=1;

CREATE INDEX IX_cand ON #candidates(cli_id, dt_rep, con_id);

/* ---------- ставка счета на ключевую дату из интервалов ---------- */
IF OBJECT_ID('tempdb..#cand_rates') IS NOT NULL DROP TABLE #cand_rates;
SELECT
  cand.cli_id, cand.con_id, cand.dt_rep,
  rate_on_date = ri.rate_con
INTO #cand_rates
FROM #candidates cand
OUTER APPLY (
  SELECT TOP (1) r.rate_con
  FROM WORK.NS_RateIntervals r
  WHERE r.con_id = cand.con_id
    AND cand.dt_rep BETWEEN r.dt_from AND r.dt_to
  ORDER BY r.dt_from DESC
) ri;

CREATE INDEX IX_candr ON #cand_rates(cli_id, dt_rep, rate_on_date DESC, con_id);

/* ---------- события перелива: EOM ---------- */
IF OBJECT_ID('tempdb..#evt_eom') IS NOT NULL DROP TABLE #evt_eom;
;WITH ranked AS (
  SELECT cr.*, rn = ROW_NUMBER() OVER(
           PARTITION BY cr.cli_id, cr.dt_rep
           ORDER BY cr.rate_on_date DESC, cr.con_id
       )
  FROM #cand_rates cr
  WHERE cr.dt_rep IN (SELECT d FROM #eom)
)
SELECT
  r.cli_id, r.con_id, r.dt_rep,
  out_rub = cs.out_rub_sum,
  reason  = CAST('EOM' AS varchar(8))
INTO #evt_eom
FROM ranked r
JOIN #cli_sum cs ON cs.cli_id=r.cli_id
WHERE r.rn=1;

CREATE INDEX IX_evte ON #evt_eom(cli_id, dt_rep);

/* ---------- события перелива: 1-е числа ---------- */
IF OBJECT_ID('tempdb..#evt_d1') IS NOT NULL DROP TABLE #evt_d1;
;WITH ranked AS (
  SELECT cr.*, rn = ROW_NUMBER() OVER(
           PARTITION BY cr.cli_id, cr.dt_rep
           ORDER BY cr.rate_on_date DESC, cr.con_id
       )
  FROM #cand_rates cr
  WHERE cr.dt_rep IN (SELECT d FROM #d1)
)
SELECT
  r.cli_id, r.con_id, r.dt_rep,
  out_rub = cs.out_rub_sum,
  reason  = CAST('D1' AS varchar(8))
INTO #evt_d1
FROM ranked r
JOIN #cli_sum cs ON cs.cli_id=r.cli_id
WHERE r.rn=1;

CREATE INDEX IX_evtd1 ON #evt_d1(cli_id, dt_rep);

/* ---------- итоговые события ---------- */
IF OBJECT_ID('WORK.NS_AllocEvents','U') IS NOT NULL DROP TABLE WORK.NS_AllocEvents;
SELECT cli_id, con_id, dt_rep, out_rub, reason
INTO WORK.NS_AllocEvents
FROM (
  SELECT * FROM #evt_eom
  UNION ALL
  SELECT * FROM #evt_d1
) u;

CREATE INDEX IX_aevt_cli_dt ON WORK.NS_AllocEvents(cli_id, dt_rep);
CREATE INDEX IX_aevt_con_dt ON WORK.NS_AllocEvents(con_id, dt_rep);

/* ---------- дневная решётка (степ-режим) ---------- */
IF OBJECT_ID('tempdb..#day_cli') IS NOT NULL DROP TABLE #day_cli;
SELECT c.d AS dt_rep, cl.cli_id
INTO #day_cli
FROM #cal c
CROSS JOIN #clients cl;

CREATE INDEX IX_daycli ON #day_cli(cli_id, dt_rep);

/* ---------- до первого события держим якорный сплит ---------- */
IF OBJECT_ID('tempdb..#first_evt') IS NOT NULL DROP TABLE #first_evt;
SELECT cli_id, first_evt = MIN(dt_rep)
INTO #first_evt
FROM WORK.NS_AllocEvents
GROUP BY cli_id;

CREATE UNIQUE INDEX IX_firstevt ON #first_evt(cli_id);

IF OBJECT_ID('tempdb..#alloc_pre') IS NOT NULL DROP TABLE #alloc_pre;
-- клиенты с событиями: дни до first_evt — сплит якоря
SELECT dc.dt_rep, ab.cli_id, ab.con_id, ab.out_rub
INTO #alloc_pre
FROM #day_cli dc
JOIN #first_evt fe ON fe.cli_id=dc.cli_id
JOIN #anchor_bal ab ON ab.cli_id=dc.cli_id
WHERE dc.dt_rep < fe.first_evt

UNION ALL
-- клиенты без событий: весь горизонт — сплит якоря
SELECT dc.dt_rep, ab.cli_id, ab.con_id, ab.out_rub
FROM #day_cli dc
JOIN #anchor_bal ab ON ab.cli_id=dc.cli_id
WHERE NOT EXISTS (SELECT 1 FROM #first_evt fe WHERE fe.cli_id=dc.cli_id);

/* ---------- после первого события — победитель на последнем ивенте ---------- */
IF OBJECT_ID('tempdb..#alloc_post') IS NOT NULL DROP TABLE #alloc_post;
SELECT
  dc.dt_rep,
  dc.cli_id,
  win.con_id,
  out_rub = cs.out_rub_sum
INTO #alloc_post
FROM #day_cli dc
JOIN #first_evt fe ON fe.cli_id=dc.cli_id AND dc.dt_rep>=fe.first_evt
JOIN #cli_sum cs ON cs.cli_id=dc.cli_id
OUTER APPLY (
  SELECT TOP (1) ae.con_id
  FROM WORK.NS_AllocEvents ae
  WHERE ae.cli_id=dc.cli_id AND ae.dt_rep<=dc.dt_rep
  ORDER BY ae.dt_rep DESC, ae.con_id
) win;

/* ---------- общий дневной пул ---------- */
IF OBJECT_ID('tempdb..#alloc_daily') IS NOT NULL DROP TABLE #alloc_daily;
SELECT * INTO #alloc_daily FROM #alloc_pre;
INSERT #alloc_daily(dt_rep, cli_id, con_id, out_rub)
SELECT dt_rep, cli_id, con_id, out_rub FROM #alloc_post;

CREATE INDEX IX_ad ON #alloc_daily(con_id, dt_rep);

/* ---------- ставка на день из интервалов ---------- */
IF OBJECT_ID('tempdb..#rates_daily') IS NOT NULL DROP TABLE #rates_daily;
SELECT
  ad.dt_rep,
  ad.con_id,
  rate_con = ri.rate_con
INTO #rates_daily
FROM #alloc_daily ad
OUTER APPLY (
  SELECT TOP (1) r.rate_con
  FROM WORK.NS_RateIntervals r
  WHERE r.con_id = ad.con_id
    AND ad.dt_rep BETWEEN r.dt_from AND r.dt_to
  ORDER BY r.dt_from DESC
) ri;

CREATE INDEX IX_rd ON #rates_daily(con_id, dt_rep);

/* ---------- итоговая агрегированная лента ---------- */
IF OBJECT_ID('WORK.Forecast_NS_Promo','U') IS NOT NULL DROP TABLE WORK.Forecast_NS_Promo;

SELECT
  ad.dt_rep,
  out_rub_total = SUM(ad.out_rub),
  rate_avg      = SUM(ad.out_rub * CAST(rd.rate_con AS decimal(9,4))) / NULLIF(SUM(ad.out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM #alloc_daily ad
JOIN #rates_daily rd
  ON rd.con_id = ad.con_id AND rd.dt_rep = ad.dt_rep
GROUP BY ad.dt_rep;

/* ---------- контрольки ---------- */
PRINT N'=== events (sample) ===';
SELECT TOP (50) * FROM WORK.NS_AllocEvents ORDER BY cli_id, dt_rep;

PRINT N'=== daily aggregate (TOP 100) ===';
SELECT TOP (100) * FROM WORK.Forecast_NS_Promo ORDER BY dt_rep;
