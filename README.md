SET NOCOUNT ON;

DECLARE @JulEOM    date = '2025-07-31';
DECLARE @ChkDate   date = '2025-08-12';
DECLARE @CloseFrom date = '2025-08-01';
DECLARE @CloseTo   date = '2025-08-11';  -- плановые закрытия

/* Мини-справочник ФУ (дешёвый join по продукту) */
IF OBJECT_ID('tempdb..#fu') IS NOT NULL DROP TABLE #fu;
CREATE TABLE #fu (prod_name_res nvarchar(255) NOT NULL PRIMARY KEY);
INSERT INTO #fu(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* 1) SEED: клиенты, у кого на 31.07 были ТОЛЬКО ФУ-вклады «Срочные»,
       и все эти ФУ закрываются 01–11.08 (не исключаем НС);
       исключаем клиентов с закрытиями НЕ-ФУ в августе. */
IF OBJECT_ID('tempdb..#seed31') IS NOT NULL DROP TABLE #seed31;
WITH rows31 AS (
  SELECT
    t.cli_id,
    t.con_id,
    is_fu    = CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END,
    dt_close = CAST(t.dt_close AS date),
    vol31    = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
  FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
  LEFT JOIN #fu f ON f.prod_name_res = t.PROD_NAME_res
  WHERE t.dt_rep = @JulEOM
    AND t.block_name = N'Привлечение ФЛ'
    AND t.od_flag = 1
    AND t.cur = '810'
    AND t.out_rub IS NOT NULL
    AND LTRIM(RTRIM(t.section_name)) = N'Срочные'
    AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  GROUP BY t.cli_id, t.con_id, CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END, t.dt_close
),
per_client AS (
  SELECT
    cli_id,
    has_nonfu_31     = MAX(CASE WHEN is_fu = 0 THEN 1 ELSE 0 END),
    fu_cnt_31        = SUM(CASE WHEN is_fu = 1 THEN 1 ELSE 0 END),
    fu_cnt_close_aug = SUM(CASE WHEN is_fu = 1 AND dt_close BETWEEN @CloseFrom AND @CloseTo THEN 1 ELSE 0 END),
    vol31_fu_total   = SUM(CASE WHEN is_fu = 1 THEN vol31 ELSE 0 END)
  FROM rows31
  GROUP BY cli_id
),
nonfu_close_aug AS (  -- у кого в августе закрывались НЕ-ФУ «Срочные» — исключим
  SELECT DISTINCT t.cli_id
  FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
  LEFT JOIN #fu f ON f.prod_name_res = t.PROD_NAME_res
  WHERE CAST(t.dt_close AS date) BETWEEN @CloseFrom AND @CloseTo
    AND f.prod_name_res IS NULL
    AND t.block_name = N'Привлечение ФЛ'
    AND t.od_flag = 1
    AND t.cur = '810'
    AND LTRIM(RTRIM(t.section_name)) = N'Срочные'
    AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
)
SELECT pc.cli_id,
       pc.vol31_fu_total AS vol_deposits_close31
INTO #seed31
FROM per_client pc
LEFT JOIN nonfu_close_aug x ON x.cli_id = pc.cli_id
WHERE pc.has_nonfu_31 = 0              -- на 31.07 только ФУ (из «Срочные»)
  AND pc.fu_cnt_31 > 0
  AND pc.fu_cnt_close_aug = pc.fu_cnt_31  -- все ФУ закрываются 01–11.08
  AND x.cli_id IS NULL;                   -- нет закрытий НЕ-ФУ в августе

CREATE UNIQUE CLUSTERED INDEX IX_seed_cli ON #seed31(cli_id);

/* 2) НС (накопительный счёт) на 31.07 и на 12.08 для тех же клиентов */
IF OBJECT_ID('tempdb..#ns31') IS NOT NULL DROP TABLE #ns31;
SELECT
  t.cli_id,
  vol_ns31 = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #ns31
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id = t.cli_id
WHERE t.dt_rep = @JulEOM
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub IS NOT NULL
  AND LTRIM(RTRIM(t.section_name)) = N'Накопительный счёт'
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.cli_id;

CREATE UNIQUE CLUSTERED INDEX IX_ns31_cli ON #ns31(cli_id);

IF OBJECT_ID('tempdb..#ns12') IS NOT NULL DROP TABLE #ns12;
SELECT
  t.cli_id,
  vol_ns12 = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #ns12
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id = t.cli_id
WHERE t.dt_rep = @ChkDate
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub IS NOT NULL
  AND LTRIM(RTRIM(t.section_name)) = N'Накопительный счёт'
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.cli_id;

CREATE UNIQUE CLUSTERED INDEX IX_ns12_cli ON #ns12(cli_id);

/* 3) Открытия вкладов 01–12.08 (по состоянию на 12.08) — «Срочные» */
IF OBJECT_ID('tempdb..#open_dep') IS NOT NULL DROP TABLE #open_dep;
SELECT
  t.cli_id,
  opened_fu   = MAX(CASE WHEN f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END),
  opened_bank = MAX(CASE WHEN f.prod_name_res IS NULL     THEN 1 ELSE 0 END),
  vol_fu_1208   = SUM(CASE WHEN f.prod_name_res IS NOT NULL AND t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END),
  vol_bank_1208 = SUM(CASE WHEN f.prod_name_res IS NULL     AND t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #open_dep
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id = t.cli_id
LEFT JOIN #fu  f ON f.prod_name_res = t.PROD_NAME_res
WHERE t.dt_rep = @ChkDate
  AND t.dt_open BETWEEN @CloseFrom AND @ChkDate
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub IS NOT NULL
  AND LTRIM(RTRIM(t.section_name)) = N'Срочные'
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.cli_id;

CREATE UNIQUE CLUSTERED INDEX IX_open_cli ON #open_dep(cli_id);

/* -------------------- ПРИНТЫ -------------------- */

/* ПРИНТ 1. SEED: «у клиентов закрываются вклады» — клиенты, объём вкладов, объём НС (на 31.07) */
SELECT
  'SEED: 31.07 только ФУ (все закрытия 01–11.08)' AS label,
  COUNT(*) AS clients_cnt,
  SUM(s.vol_deposits_close31) AS deposits_volume_31_close,
  SUM(ISNULL(ns31.vol_ns31, 0)) AS ns_volume_31
FROM #seed31 s
LEFT JOIN #ns31 ns31 ON ns31.cli_id = s.cli_id;

/* ПРИНТ 2. 12.08 — открыли ТОЛЬКО БАНК: клиенты, объём вкладов, объём НС (на 12.08) */
SELECT
  'OPEN 01–12.08: только Банк' AS label,
  COUNT(*) AS clients_cnt,
  SUM(od.vol_bank_1208) AS deposits_volume_1208,
  SUM(ISNULL(ns12.vol_ns12, 0)) AS ns_volume_1208
FROM #open_dep od
LEFT JOIN #ns12 ns12 ON ns12.cli_id = od.cli_id
WHERE od.opened_bank = 1 AND od.opened_fu = 0;

/* ПРИНТ 3. 12.08 — открыли ТОЛЬКО ФУ: клиенты, объём вкладов, объём НС (на 12.08) */
SELECT
  'OPEN 01–12.08: только ФУ' AS label,
  COUNT(*) AS clients_cnt,
  SUM(od.vol_fu_1208) AS deposits_volume_1208,
  SUM(ISNULL(ns12.vol_ns12, 0)) AS ns_volume_1208
FROM #open_dep od
LEFT JOIN #ns12 ns12 ON ns12.cli_id = od.cli_id
WHERE od.opened_fu = 1 AND od.opened_bank = 0;

/* ПРИНТ 4. 12.08 — открыли ФУ И БАНК: клиенты, общий объём вкладов, объём НС (на 12.08) */
SELECT
  'OPEN 01–12.08: ФУ и Банк' AS label,
  COUNT(*) AS clients_cnt,
  SUM(od.vol_fu_1208 + od.vol_bank_1208) AS deposits_volume_1208,
  SUM(ISNULL(ns12.vol_ns12, 0)) AS ns_volume_1208
FROM #open_dep od
LEFT JOIN #ns12 ns12 ON ns12.cli_id = od.cli_id
WHERE od.opened_fu = 1 AND od.opened_bank = 1;

/* ПРИНТ 5. 12.08 — НЕ открывали вклад, но ОТКРЫЛИ/ПОПОЛНИЛИ НС:
   клиенты, НС был, НС стал, изменение НС */
;WITH no_dep_open AS (
  SELECT s.cli_id
  FROM #seed31 s
  LEFT JOIN #open_dep od ON od.cli_id = s.cli_id
  WHERE od.cli_id IS NULL   -- нет открытий вкладов 01–12.08
),
ns_delta AS (
  SELECT
    n.cli_id,
    ns31 = ISNULL(ns31.vol_ns31, 0),
    ns12 = ISNULL(ns12.vol_ns12, 0),
    delta = ISNULL(ns12.vol_ns12, 0) - ISNULL(ns31.vol_ns31, 0)
  FROM no_dep_open n
  LEFT JOIN #ns31 ns31 ON ns31.cli_id = n.cli_id
  LEFT JOIN #ns12 ns12 ON ns12.cli_id = n.cli_id
)
SELECT
  'NO DEPOSIT OPENS 01–12.08, BUT NS INCREASED' AS label,
  COUNT(*) AS clients_cnt,
  SUM(ns31) AS ns_volume_was_31,
  SUM(ns12) AS ns_volume_now_1208,
  SUM(delta) AS ns_change
FROM ns_delta
WHERE delta > 0;

/* ПРИНТ 6. Остальные клиенты («ушли»): не открывали вклады и НС не вырос
   (НС либо не было, либо снизился/нулевой прирост) — выводим только кол-во */
;WITH no_dep_open AS (
  SELECT s.cli_id
  FROM #seed31 s
  LEFT JOIN #open_dep od ON od.cli_id = s.cli_id
  WHERE od.cli_id IS NULL
),
ns_delta AS (
  SELECT
    n.cli_id,
    delta = ISNULL(ns12.vol_ns12, 0) - ISNULL(ns31.vol_ns31, 0)
  FROM no_dep_open n
  LEFT JOIN #ns31 ns31 ON ns31.cli_id = n.cli_id
  LEFT JOIN #ns12 ns12 ON ns12.cli_id = n.cli_id
)
SELECT
  'LEFT: no deposit opens & NS not increased' AS label,
  COUNT(*) AS clients_cnt
FROM ns_delta
WHERE delta <= 0;
