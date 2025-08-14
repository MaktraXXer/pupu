SET NOCOUNT ON;

DECLARE @JulEOM    date = '2025-07-31';
DECLARE @ChkDate   date = '2025-08-12';
DECLARE @CloseFrom date = '2025-08-01';
DECLARE @CloseTo   date = '2025-08-11';  -- плановые закрытия

/* мини-справочник ФУ (для дешёвого join'а) */
IF OBJECT_ID('tempdb..#fu') IS NOT NULL DROP TABLE #fu;
CREATE TABLE #fu (prod_name_res nvarchar(255) NOT NULL PRIMARY KEY);
INSERT INTO #fu(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* 1) SEED: клиенты, у кого на 31.07 были ТОЛЬКО ФУ (вклады "Срочные"),
       и ВСЕ эти ФУ закрываются 01–11.08; исключаем клиентов с закрытиями не-ФУ в августе */
IF OBJECT_ID('tempdb..#seed31') IS NOT NULL DROP TABLE #seed31;
WITH rows31 AS (
  SELECT
    t.cli_id, t.con_id,
    is_fu = CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END,
    dt_close = CAST(t.dt_close AS date),
    vol31 = SUM(CASE WHEN t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
  FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
  LEFT JOIN #fu f ON f.prod_name_res = t.PROD_NAME_res
  WHERE t.dt_rep = @JulEOM
    AND t.block_name = N'Привлечение ФЛ' AND t.od_flag=1 AND t.cur='810'
    AND t.out_rub IS NOT NULL
    AND t.section_name IN (N'Срочные', N'Срочные ')
    AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  GROUP BY t.cli_id, t.con_id, CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END, t.dt_close
),
per_client AS (
  SELECT
    cli_id,
    has_nonfu_31     = MAX(CASE WHEN is_fu=0 THEN 1 ELSE 0 END),
    fu_cnt_31        = SUM(CASE WHEN is_fu=1 THEN 1 ELSE 0 END),
    fu_cnt_close_aug = SUM(CASE WHEN is_fu=1 AND dt_close BETWEEN @CloseFrom AND @CloseTo THEN 1 ELSE 0 END),
    vol31_fu_total   = SUM(CASE WHEN is_fu=1 THEN vol31 ELSE 0 END)
  FROM rows31
  GROUP BY cli_id
),
nonfu_close_aug AS (  -- исключаем, если у клиента были закрытия не-ФУ в августе
  SELECT DISTINCT t.cli_id
  FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
  LEFT JOIN #fu f ON f.prod_name_res = t.PROD_NAME_res
  WHERE CAST(t.dt_close AS date) BETWEEN @CloseFrom AND @CloseTo
    AND f.prod_name_res IS NULL
    AND t.block_name = N'Привлечение ФЛ' AND t.od_flag=1 AND t.cur='810'
    AND t.section_name IN (N'Срочные', N'Срочные ')
    AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
)
SELECT pc.cli_id, pc.vol31_fu_total AS vol_closed_31
INTO #seed31
FROM per_client pc
LEFT JOIN nonfu_close_aug x ON x.cli_id = pc.cli_id
WHERE pc.has_nonfu_31 = 0
  AND pc.fu_cnt_31>0
  AND pc.fu_cnt_close_aug = pc.fu_cnt_31
  AND x.cli_id IS NULL;

CREATE UNIQUE CLUSTERED INDEX IX_seed_cli ON #seed31(cli_id);

/* 2) НС на 31.07 и 12.08 (по тем же клиентам) */
IF OBJECT_ID('tempdb..#ns31') IS NOT NULL DROP TABLE #ns31;
SELECT
  t.cli_id,
  vol_ns31 = SUM(CASE WHEN t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #ns31
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id=t.cli_id
WHERE t.dt_rep = @JulEOM
  AND t.block_name = N'Привлечение ФЛ' AND t.od_flag=1 AND t.cur='810'
  AND t.out_rub IS NOT NULL
  AND t.section_name = N'Накопительный счёт'
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.cli_id;
CREATE UNIQUE CLUSTERED INDEX IX_ns31_cli ON #ns31(cli_id);

IF OBJECT_ID('tempdb..#ns12_con') IS NOT NULL DROP TABLE #ns12_con;
SELECT
  t.con_id,
  t.cli_id,
  dt_open = CAST(t.dt_open AS date),
  vol_ns12 = SUM(CASE WHEN t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #ns12_con
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id=t.cli_id
WHERE t.dt_rep = @ChkDate
  AND t.block_name = N'Привлечение ФЛ' AND t.od_flag=1 AND t.cur='810'
  AND t.out_rub IS NOT NULL
  AND t.section_name = N'Накопительный счёт'
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.con_id, t.cli_id, t.dt_open;
CREATE UNIQUE CLUSTERED INDEX IX_ns12_con ON #ns12_con(con_id);

-- агрегаты по НС на 12.08 и притокам
IF OBJECT_ID('tempdb..#ns12') IS NOT NULL DROP TABLE #ns12;
SELECT cli_id, SUM(vol_ns12) AS vol_ns12
INTO #ns12
FROM #ns12_con
GROUP BY cli_id;
CREATE UNIQUE CLUSTERED INDEX IX_ns12_cli ON #ns12(cli_id);

IF OBJECT_ID('tempdb..#ns_open') IS NOT NULL DROP TABLE #ns_open;
SELECT cli_id,
       vol_ns_open = SUM(CASE WHEN dt_open BETWEEN @CloseFrom AND @ChkDate THEN vol_ns12 ELSE 0 END)
INTO #ns_open
FROM #ns12_con
GROUP BY cli_id;
CREATE UNIQUE CLUSTERED INDEX IX_nsopen_cli ON #ns_open(cli_id);

/* «пополнения НС»: прирост по уже существующим НС (только положительный дельта) */
IF OBJECT_ID('tempdb..#ns31_con') IS NOT NULL DROP TABLE #ns31_con;
SELECT t.con_id, t.cli_id,
       vol_ns31 = SUM(CASE WHEN t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #ns31_con
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id=t.cli_id
WHERE t.dt_rep = @JulEOM
  AND t.block_name = N'Привлечение ФЛ' AND t.od_flag=1 AND t.cur='810'
  AND t.out_rub IS NOT NULL
  AND t.section_name = N'Накопительный счёт'
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.con_id, t.cli_id;
CREATE UNIQUE CLUSTERED INDEX IX_ns31_con2 ON #ns31_con(con_id);

IF OBJECT_ID('tempdb..#ns_refill') IS NOT NULL DROP TABLE #ns_refill;
SELECT n12.cli_id,
       vol_ns_refill = SUM(CASE WHEN (n12.vol_ns12 - ISNULL(n31.vol_ns31,0)) > 0
                                THEN n12.vol_ns12 - ISNULL(n31.vol_ns31,0) ELSE 0 END)
INTO #ns_refill
FROM #ns12_con n12
LEFT JOIN #ns31_con n31 ON n31.con_id = n12.con_id
GROUP BY n12.cli_id;
CREATE UNIQUE CLUSTERED INDEX IX_nsrefill_cli ON #ns_refill(cli_id);

/* 3) ОТКРЫТИЯ 01–12.08 по вкладам: классификация клиента и объёмы на 12.08 */
IF OBJECT_ID('tempdb..#open_aug') IS NOT NULL DROP TABLE #open_aug;
SELECT
  t.cli_id,
  opened_fu   = MAX(CASE WHEN f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END),
  opened_bank = MAX(CASE WHEN f.prod_name_res IS NULL     THEN 1 ELSE 0 END),
  vol_fu      = SUM(CASE WHEN f.prod_name_res IS NOT NULL AND t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END),
  vol_bank    = SUM(CASE WHEN f.prod_name_res IS NULL     AND t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #open_aug
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id = t.cli_id
LEFT JOIN #fu  f ON f.prod_name_res = t.PROD_NAME_res
WHERE t.dt_rep = @ChkDate
  AND t.dt_open BETWEEN @CloseFrom AND @ChkDate
  AND t.block_name = N'Привлечение ФЛ' AND t.od_flag=1 AND t.cur='810'
  AND t.out_rub IS NOT NULL
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.cli_id;
CREATE UNIQUE CLUSTERED INDEX IX_open_cli ON #open_aug(cli_id);

/* ===== ПРИНТ 1: SEED — сколько клиентов, объём закрывающихся вкладов, и НС на 31.07 ===== */
SELECT
  'SEED: 31.07 только ФУ (все закрытия 01–11.08)' AS label,
  COUNT(*) AS clients_cnt,
  SUM(s.vol_closed_31) AS vol_deposits_close31,
  SUM(ISNULL(n31.vol_ns31,0)) AS vol_ns_31
FROM #seed31 s
LEFT JOIN #ns31 n31 ON n31.cli_id = s.cli_id;

/* ===== ПРИНТ 2: OPEN только БАНК + изменение НС и приток в НС ===== */
SELECT
  'OPEN: только Банк (01–12.08)' AS label,
  COUNT(*) AS clients_cnt,
  SUM(oa.vol_bank) AS vol_deposits_1208,
  SUM(ISNULL(n12.vol_ns12,0) - ISNULL(n31.vol_ns31,0)) AS ns_change_total,
  SUM(ISNULL(nope.vol_ns_open,0) + ISNULL(nref.vol_ns_refill,0)) AS ns_positive_inflow
FROM #open_aug oa
LEFT JOIN #ns31 n31   ON n31.cli_id = oa.cli_id
LEFT JOIN #ns12 n12   ON n12.cli_id = oa.cli_id
LEFT JOIN #ns_open nope ON nope.cli_id = oa.cli_id
LEFT JOIN #ns_refill nref ON nref.cli_id = oa.cli_id
WHERE oa.opened_bank=1 AND oa.opened_fu=0;

/* ===== ПРИНТ 3: OPEN только ФУ + изменение НС и приток в НС ===== */
SELECT
  'OPEN: только ФУ (01–12.08)' AS label,
  COUNT(*) AS clients_cnt,
  SUM(oa.vol_fu) AS vol_deposits_1208,
  SUM(ISNULL(n12.vol_ns12,0) - ISNULL(n31.vol_ns31,0)) AS ns_change_total,
  SUM(ISNULL(nope.vol_ns_open,0) + ISNULL(nref.vol_ns_refill,0)) AS ns_positive_inflow
FROM #open_aug oa
LEFT JOIN #ns31 n31   ON n31.cli_id = oa.cli_id
LEFT JOIN #ns12 n12   ON n12.cli_id = oa.cli_id
LEFT JOIN #ns_open nope ON nope.cli_id = oa.cli_id
LEFT JOIN #ns_refill nref ON nref.cli_id = oa.cli_id
WHERE oa.opened_fu=1 AND oa.opened_bank=0;

/* ===== ПРИНТ 4: OPEN ФУ и БАНК (объёмы раздельно) + НС ===== */
SELECT
  'OPEN: ФУ и Банк (01–12.08)' AS label,
  COUNT(*) AS clients_cnt,
  SUM(oa.vol_fu)   AS vol_deposits_fu_1208,
  SUM(oa.vol_bank) AS vol_deposits_bank_1208,
  SUM(ISNULL(n12.vol_ns12,0) - ISNULL(n31.vol_ns31,0)) AS ns_change_total,
  SUM(ISNULL(nope.vol_ns_open,0) + ISNULL(nref.vol_ns_refill,0)) AS ns_positive_inflow
FROM #open_aug oa
LEFT JOIN #ns31 n31   ON n31.cli_id = oa.cli_id
LEFT JOIN #ns12 n12   ON n12.cli_id = oa.cli_id
LEFT JOIN #ns_open nope ON nope.cli_id = oa.cli_id
LEFT JOIN #ns_refill nref ON nref.cli_id = oa.cli_id
WHERE oa.opened_fu=1 AND oa.opened_bank=1;

/* ===== ПРИНТ 5: NO OPENS (ничего не открыли) + изменение НС и приток НС ===== */
;WITH no_open_cli AS (
  SELECT s.cli_id
  FROM #seed31 s
  LEFT JOIN #open_aug oa ON oa.cli_id = s.cli_id
  WHERE oa.cli_id IS NULL
)
SELECT
  'NO OPENS (01–12.08)' AS label,
  COUNT(*) AS clients_cnt,
  CAST(0 AS decimal(20,2)) AS vol_deposits_1208,
  SUM(ISNULL(n12.vol_ns12,0) - ISNULL(n31.vol_ns31,0)) AS ns_change_total,
  SUM(ISNULL(nope.vol_ns_open,0) + ISNULL(nref.vol_ns_refill,0)) AS ns_positive_inflow
FROM no_open_cli x
LEFT JOIN #ns31 n31   ON n31.cli_id = x.cli_id
LEFT JOIN #ns12 n12   ON n12.cli_id = x.cli_id
LEFT JOIN #ns_open nope ON nope.cli_id = x.cli_id
LEFT JOIN #ns_refill nref ON nref.cli_id = x.cli_id;
