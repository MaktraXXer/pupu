/* ================================================================
   NS-forecast — Part 2 (ALLOC)
   Требует результаты Part 1:
     1) WORK.NS_RatesKeyDates(con_id, cli_id, dt_key, rate_con, is_firstday, is_eom)
     2) WORK.NS_BalPromoAnchor(con_id, cli_id, out_rub [, TSEGMENTNAME])
   Правила:
     • EOM и 1-е числа — полный перелив Σ клиента на счёт-победитель
       (максимальная ставка на эту дату по его счетам; тай-брейк: con_id).
     • До первого события — держим якорный сплит по всем счетам клиента.
     • После любого события — держим всю сумму клиента на последнем победителе.
     • Ставка на день по счёту — последняя ключевая ставка ≤ дата.
   Итог:
     • WORK.NS_AllocEvents — события перелива (только EOM и 1-е).
     • WORK.Forecast_NS_Promo — дневной агрегат (Σ объёма, средневзвешенная ставка).
   ================================================================ */
SET NOCOUNT ON;

/* ----------------- ПАРАМЕТРЫ/САНИТИ И ДАТЫ ------------------- */
IF OBJECT_ID('WORK.NS_RatesKeyDates','U') IS NULL
  RAISERROR('WORK.NS_RatesKeyDates is required (from Part 1).',16,1);

IF OBJECT_ID('WORK.NS_BalPromoAnchor','U') IS NULL
  RAISERROR('WORK.NS_BalPromoAnchor is required (from Part 1).',16,1);

DECLARE @StartDate date, @EndDate date;

SELECT
  @StartDate = MIN(dt_key),
  @EndDate   = MAX(dt_key)
FROM WORK.NS_RatesKeyDates;

IF @StartDate IS NULL OR @EndDate IS NULL
  RAISERROR('NS_RatesKeyDates has no dates.',16,1);

/* --------------------- БАЗОВЫЕ СЕТЫ -------------------------- */
IF OBJECT_ID('tempdb..#rates_key') IS NOT NULL DROP TABLE #rates_key;
SELECT
  con_id     = CAST(con_id AS bigint),
  cli_id     = CAST(cli_id AS bigint),
  dt_key     = CAST(dt_key AS date),
  rate_con   = CAST(rate_con AS decimal(9,4)),
  is_firstday= CAST(COALESCE(is_firstday, CASE WHEN DAY(dt_key)=1 THEN 1 ELSE 0 END) AS bit),
  is_eom     = CAST(COALESCE(is_eom, CASE WHEN dt_key = EOMONTH(dt_key) THEN 1 ELSE 0 END) AS bit)
INTO #rates_key
FROM WORK.NS_RatesKeyDates;

CREATE INDEX IX_rk_1 ON #rates_key(con_id, dt_key);
CREATE INDEX IX_rk_2 ON #rates_key(cli_id, dt_key);

IF OBJECT_ID('tempdb..#anchor_bal') IS NOT NULL DROP TABLE #anchor_bal;
SELECT
  con_id   = CAST(con_id AS bigint),
  cli_id   = CAST(cli_id AS bigint),
  out_rub  = CAST(out_rub AS decimal(20,2))
INTO #anchor_bal
FROM WORK.NS_BalPromoAnchor;

CREATE INDEX IX_ab_1 ON #anchor_bal(cli_id, con_id);

IF OBJECT_ID('tempdb..#clients') IS NOT NULL DROP TABLE #clients;
SELECT DISTINCT cli_id INTO #clients FROM #anchor_bal;
CREATE UNIQUE INDEX IX_cli ON #clients(cli_id);

IF OBJECT_ID('tempdb..#cli_sum') IS NOT NULL DROP TABLE #cli_sum;
SELECT cli_id, out_rub_sum = SUM(out_rub)
INTO #cli_sum
FROM #anchor_bal
GROUP BY cli_id;
CREATE UNIQUE INDEX IX_clisum ON #cli_sum(cli_id);

/* --------------------- КАЛЕНДАРЬ ДНЕЙ ------------------------ */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
;WITH span AS (
  SELECT d = @StartDate
  UNION ALL
  SELECT DATEADD(day,1,d) FROM span WHERE DATEADD(day,1,d) <= @EndDate
)
SELECT d INTO #cal FROM span OPTION (MAXRECURSION 0);
CREATE UNIQUE INDEX IX_cal ON #cal(d);

/* Ключевые даты из Part 1 (на всякий случай) */
IF OBJECT_ID('tempdb..#d_eom') IS NOT NULL DROP TABLE #d_eom;
SELECT DISTINCT d = rk.dt_key
INTO #d_eom
FROM #rates_key rk
WHERE rk.is_eom = 1;

IF OBJECT_ID('tempdb..#d_first') IS NOT NULL DROP TABLE #d_first;
SELECT DISTINCT d = rk.dt_key
INTO #d_first
FROM #rates_key rk
WHERE rk.is_firstday = 1;

/* ------------------ РЕЙТЫ НА КЛЮЧЕВЫЕ ДАТЫ ------------------ */
/* helper: ставка счёта на заданную дату = последняя ключевая ≤ дата */
-- Нужна только для подбора победителя на EOM/1-е
IF OBJECT_ID('tempdb..#acc_candidates') IS NOT NULL DROP TABLE #acc_candidates;
/* Кандидаты: для каждого клиента — все его счета (из якоря) × ключевые даты */
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

CREATE INDEX IX_cand ON #acc_candidates(cli_id, dt_rep, con_id);

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

CREATE INDEX IX_arok ON #acc_rates_on_key(cli_id, dt_rep, rate_on_date DESC, con_id);

/* -------------------- СОБЫТИЯ ПЕРЕЛИВА ----------------------- */
/* Победитель на EOM */
IF OBJECT_ID('tempdb..#evt_base') IS NOT NULL DROP TABLE #evt_base;
;WITH ranked AS (
  SELECT
    a.cli_id, a.dt_rep,
    a.con_id,
    a.rate_on_date,
    rn = ROW_NUMBER() OVER (
          PARTITION BY a.cli_id, a.dt_rep
          ORDER BY a.rate_on_date DESC, a.con_id
        )
  FROM #acc_rates_on_key a
  WHERE a.dt_rep IN (SELECT d FROM #d_eom)
)
SELECT
  e.cli_id,
  e.con_id,
  e.dt_rep,
  out_rub = cs.out_rub_sum,
  reason  = CAST('EOM' AS varchar(16))
INTO #evt_base
FROM ranked e
JOIN #cli_sum cs ON cs.cli_id = e.cli_id
WHERE e.rn = 1;

CREATE INDEX IX_evtb ON #evt_base(cli_id, dt_rep);

/* Победитель на 1-е числа */
IF OBJECT_ID('tempdb..#evt_first') IS NOT NULL DROP TABLE #evt_first;
;WITH ranked AS (
  SELECT
    a.cli_id, a.dt_rep,
    a.con_id,
    a.rate_on_date,
    rn = ROW_NUMBER() OVER (
          PARTITION BY a.cli_id, a.dt_rep
          ORDER BY a.rate_on_date DESC, a.con_id
        )
  FROM #acc_rates_on_key a
  WHERE a.dt_rep IN (SELECT d FROM #d_first)
)
SELECT
  e.cli_id,
  e.con_id,
  e.dt_rep,
  out_rub = cs.out_rub_sum,
  reason  = CAST('D1' AS varchar(16))
INTO #evt_first
FROM ranked e
JOIN #cli_sum cs ON cs.cli_id = e.cli_id
WHERE e.rn = 1;

CREATE INDEX IX_evtf ON #evt_first(cli_id, dt_rep);

/* Материализуем единый список событий */
IF OBJECT_ID('WORK.NS_AllocEvents','U') IS NOT NULL DROP TABLE WORK.NS_AllocEvents;

SELECT
  cli_id,
  con_id,
  dt_rep,
  out_rub,
  reason
INTO WORK.NS_AllocEvents
FROM (
  SELECT * FROM #evt_base
  UNION ALL
  SELECT * FROM #evt_first
) x;

CREATE INDEX IX_aevt_1 ON WORK.NS_AllocEvents(cli_id, dt_rep);
CREATE INDEX IX_aevt_2 ON WORK.NS_AllocEvents(con_id, dt_rep);

/* --------------------- СУТОЧНАЯ ЛЕНТА ------------------------ */
/* 1) До первого события держим якорный сплит по счетам */
IF OBJECT_ID('tempdb..#first_evt') IS NOT NULL DROP TABLE #first_evt;
SELECT
  cli_id,
  first_evt = MIN(dt_rep)
INTO #first_evt
FROM WORK.NS_AllocEvents
GROUP BY cli_id;

CREATE UNIQUE INDEX IX_fe ON #first_evt(cli_id);

/* day × client решётка */
IF OBJECT_ID('tempdb..#day_cli') IS NOT NULL DROP TABLE #day_cli;
SELECT c.d AS dt_rep, cl.cli_id
INTO #day_cli
FROM #cal c
CROSS JOIN #clients cl;

CREATE INDEX IX_daycli ON #day_cli(cli_id, dt_rep);

/* 2) Пре-период: дни до первого события → якорный сплит */
IF OBJECT_ID('tempdb..#alloc_pre') IS NOT NULL DROP TABLE #alloc_pre;
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

/* Клиенты без событий вообще: держим сплит весь горизонт */
SELECT
  dc.dt_rep,
  ab.cli_id,
  ab.con_id,
  ab.out_rub
FROM #day_cli dc
JOIN #anchor_bal ab
  ON ab.cli_id = dc.cli_id
WHERE NOT EXISTS (
  SELECT 1 FROM #first_evt fe WHERE fe.cli_id = dc.cli_id
);

/* 3) Пост-период: дни с первого события и дальше → winner последнего события */
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

/* Объединённые аллокации на день */
IF OBJECT_ID('tempdb..#alloc_daily') IS NOT NULL DROP TABLE #alloc_daily;
SELECT * INTO #alloc_daily FROM #alloc_pre;
INSERT #alloc_daily(dt_rep, cli_id, con_id, out_rub)
SELECT dt_rep, cli_id, con_id, out_rub FROM #alloc_post;

CREATE INDEX IX_ad_1 ON #alloc_daily(dt_rep);
CREATE INDEX IX_ad_2 ON #alloc_daily(con_id, dt_rep);

/* 4) Подтягиваем ставку счёта на день: последняя ключевая ≤ дата */
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

CREATE INDEX IX_rd_1 ON #rates_daily(con_id, dt_rep);

/* 5) Финальный агрегат по дням */
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

/* ----------------------- КОНТРОЛЬКИ -------------------------- */
PRINT N'=== events (sample) ===';
SELECT TOP (100)
  ae.cli_id, ae.dt_rep, ae.con_id, ae.out_rub, ae.reason
FROM WORK.NS_AllocEvents ae
ORDER BY ae.cli_id, ae.dt_rep;

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
