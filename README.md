SET NOCOUNT ON;

DECLARE @JulEOM    date = '2025-07-31';
DECLARE @ChkDate   date = '2025-08-12';
DECLARE @CloseFrom date = '2025-08-01';   -- плановые закрытия
DECLARE @CloseTo   date = '2025-08-11';

/* мини-справочник ФУ — дешёвый join */
IF OBJECT_ID('tempdb..#fu') IS NOT NULL DROP TABLE #fu;
CREATE TABLE #fu (prod_name_res nvarchar(255) NOT NULL PRIMARY KEY);
INSERT INTO #fu(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* 1) #seed31: клиенты, у кого на 31.07 были ТОЛЬКО ФУ и все эти ФУ закрываются 01–11.08 */
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
pc AS (
  SELECT
    cli_id,
    has_nonfu_31     = MAX(CASE WHEN is_fu=0 THEN 1 ELSE 0 END),
    fu_cnt_31        = SUM(CASE WHEN is_fu=1 THEN 1 ELSE 0 END),
    fu_cnt_close_aug = SUM(CASE WHEN is_fu=1 AND dt_close BETWEEN @CloseFrom AND @CloseTo THEN 1 ELSE 0 END),
    vol31_all        = SUM(vol31)
  FROM rows31
  GROUP BY cli_id
)
SELECT cli_id, vol_closed_31 = vol31_all
INTO #seed31
FROM pc
WHERE has_nonfu_31=0             -- на 31.07 только ФУ
  AND fu_cnt_31>0
  AND fu_cnt_close_aug=fu_cnt_31;-- и все они закрываются 01–11.08

CREATE UNIQUE CLUSTERED INDEX IX_seed_cli ON #seed31(cli_id);

/* ===== ПРИНТ 1: база — сколько клиентов и какой объём закрывался ===== */
SELECT
  'SEED: 31.07 только ФУ, все закрытия 01–11.08' AS label,
  COUNT(*)  AS clients_cnt,
  SUM(vol_closed_31) AS volume_rub
FROM #seed31;

/* 2) Открытия 01–12.08 по состоянию на 12.08 (смотрим баланс на 12.08) */
IF OBJECT_ID('tempdb..#open_aug') IS NOT NULL DROP TABLE #open_aug;
SELECT
  t.cli_id,
  opened_fu   = MAX(CASE WHEN f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END),
  opened_bank = MAX(CASE WHEN f.prod_name_res IS NULL THEN 1 ELSE 0 END),
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

/* ===== ПРИНТ 2: открылись ТОЛЬКО БАНК (01–12.08) ===== */
SELECT
  'OPEN: только Банк (01–12.08)' AS label,
  COUNT(*) AS clients_cnt,
  SUM(vol_bank) AS volume_rub
FROM #open_aug
WHERE opened_bank=1 AND opened_fu=0;

/* ===== ПРИНТ 3: открылись ТОЛЬКО ФУ (01–12.08) ===== */
SELECT
  'OPEN: только ФУ (01–12.08)' AS label,
  COUNT(*) AS clients_cnt,
  SUM(vol_fu) AS volume_rub
FROM #open_aug
WHERE opened_fu=1 AND opened_bank=0;

/* ===== ПРИНТ 4: открылись ФУ И БАНК (01–12.08), объёмы раздельно ===== */
SELECT
  'OPEN: ФУ и Банк (01–12.08)' AS label,
  COUNT(*) AS clients_cnt,
  SUM(vol_fu)   AS volume_fu_rub,
  SUM(vol_bank) AS volume_bank_rub
FROM #open_aug
WHERE opened_fu=1 AND opened_bank=1;

/* ===== ПРИНТ 5: ничего не открыли (01–12.08) ===== */
SELECT
  'NO OPENS (01–12.08)' AS label,
  (SELECT COUNT(*) FROM #seed31) - ISNULL((SELECT COUNT(*) FROM #open_aug WHERE opened_fu=1 OR opened_bank=1),0) AS clients_cnt;
