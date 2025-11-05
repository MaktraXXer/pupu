/* ================================================================
   NS-forecast — Part 2 (ALLOC)  — версия без is_firstday / is_eom
   Зависимости (из Part 1):
     • WORK.NS_RatesKeyDates(con_id, cli_id, dt_key, rate_con)
       — содержит ТОЛЬКО якорь, все EOM и все 1-е числа по каждому счёту
     • WORK.NS_BalPromoAnchor(con_id, cli_id, out_rub [, TSEGMENTNAME])
       — фактический сплит на якоре
   Правила:
     • Пересчёт победителя строго на ключевых датах:
         — EOM: базовый день, полный перелив Σ клиента на счёт с max ставкой;
         — 1-е: полный перелив Σ клиента на счёт с max ставкой;
       тай-брейк: con_id.
     • До первого события — держим якорный сплит.
     • После события — весь Σ клиента на последнем победителе.
     • Ставка счёта на любой день — последняя ключевая ставка ≤ дата.
   Итоги:
     • WORK.NS_AllocEvents — события перелива (только EOM/1-е).
     • WORK.Forecast_NS_Promo — дневной агрегат (Σ объёма, средневзвешенная ставка).
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

/* -------- ключевые даты: EOM и 1-е (вычисляем из dt_key) ----- */
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

/* --------- календарь для суточной ленты (степ-режим) -------- */
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
-- клиенты, у которых есть события: дни до first_evt -> держим сплит
SELECT
  dc.dt_rep,
  ab.cli_id,
  ab.con_id,
  ab.out_rub
INTO #alloc_pre
FROM #day_cli dc
JOIN #first_evt fe
  ON fe.cli_id = dc.cli_id
JOIN #anchor_bal ab
  ON ab.cli_id = dc.cli_id
WHERE dc.dt_rep < fe.first_evt

UNION ALL
-- клиенты без событий: весь горизонт держим сплит
SELECT
  dc.dt_rep,
  ab.cli_id,
  ab.con_id,
  ab.out_rub
FROM #day_cli dc
JOIN #anchor_bal ab
  ON ab.cli_id = dc.cli_id
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
JOIN #first_evt fe
  ON fe.cli_id = dc.cli_id
  AND dc.dt_rep >= fe.first_evt
JOIN #cli_sum cs
  ON cs.cli_id = dc.cli_id
OUTER APPLY (
  SELECT TOP (1) ae.con_id, ae.dt_rep
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

/* ------------------- контрольные выборки --------------------- */
PRINT N'=== events (sample) ===';
SELECT TOP (100) cli_id, dt_rep, con_id, out_rub, reason
FROM WORK.NS_AllocEvents
ORDER BY cli_id, dt_rep;

PRINT N'=== daily aggregate (TOP 200) ===';
SELECT TOP (200) *
FROM WORK.Forecast_NS_Promo
ORDER BY dt_rep;

PRINT N'=== Σ объёма по дням vs Anchor ===';
DECLARE @AnchorTotal decimal(20,2);
SELECT @AnchorTotal = SUM(out_rub) FROM #anchor_bal;

SELECT TOP (200)
  f.dt_rep,
  f.out_rub_total,
  diff_vs_anchor = CAST(f.out_rub_total - @AnchorTotal AS decimal(20,2)),
  f.rate_avg
FROM WORK.Forecast_NS_Promo f
ORDER BY f.dt_rep;
