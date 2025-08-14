SET NOCOUNT ON;

DECLARE @JulEOM    date = '2025-07-31';
DECLARE @ChkDate   date = '2025-08-12';
DECLARE @CloseFrom date = '2025-08-01';
DECLARE @CloseTo   date = '2025-08-11';  -- плановые 1–11 августа

/* 0) Крошечный справочник ФУ → дешёвый join */
IF OBJECT_ID('tempdb..#fu') IS NOT NULL DROP TABLE #fu;
CREATE TABLE #fu (prod_name_res nvarchar(255) NOT NULL PRIMARY KEY);
INSERT INTO #fu(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* 1) Снимок 31.07: договоры (нужны vol и dt_close) */
IF OBJECT_ID('tempdb..#j31') IS NOT NULL DROP TABLE #j31;
SELECT
    t.con_id,
    t.cli_id,
    is_fu    = CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END,
    dt_close = CAST(t.dt_close AS date),
    vol31    = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #j31
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
LEFT JOIN #fu f ON f.prod_name_res = t.PROD_NAME_res
WHERE t.dt_rep       = @JulEOM
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.con_id, t.cli_id, CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END, t.dt_close;

CREATE CLUSTERED INDEX IX_j31_cli ON #j31(cli_id);

/* 2) «Чистые» клиенты:
      — на 31.07 нет не-ФУ (только ФУ),
      — все ФУ из 31.07 закрываются 01–11.08,
      — в августе нет закрытий не-ФУ вообще. */
IF OBJECT_ID('tempdb..#eligible') IS NOT NULL DROP TABLE #eligible;
WITH per_client AS (
  SELECT
      j.cli_id,
      has_nonfu_31     = MAX(CASE WHEN j.is_fu=0 THEN 1 ELSE 0 END),
      fu_cnt_31        = SUM(CASE WHEN j.is_fu=1 THEN 1 ELSE 0 END),
      fu_cnt_close_aug = SUM(CASE WHEN j.is_fu=1 AND j.dt_close BETWEEN @CloseFrom AND @CloseTo THEN 1 ELSE 0 END)
  FROM #j31 j
  GROUP BY j.cli_id
),
clients31 AS (
  SELECT DISTINCT cli_id FROM #j31
),
nonfu_close_aug AS (   -- исключаем клиентов с закрытиями не-ФУ в августе
  SELECT DISTINCT t.cli_id
  FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
  JOIN clients31 c ON c.cli_id = t.cli_id
  LEFT JOIN #fu f ON f.prod_name_res = t.PROD_NAME_res
  WHERE CAST(t.dt_close AS date) BETWEEN @CloseFrom AND @CloseTo
    AND f.prod_name_res IS NULL
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag      = 1
    AND t.cur          = '810'
    AND t.section_name IN (N'Срочные', N'Срочные ')
    AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
)
SELECT pc.cli_id
INTO #eligible
FROM per_client pc
LEFT JOIN nonfu_close_aug x ON x.cli_id = pc.cli_id
WHERE pc.has_nonfu_31 = 0                 -- на 31.07 был только ФУ
  AND pc.fu_cnt_31 > 0
  AND pc.fu_cnt_close_aug = pc.fu_cnt_31  -- все ФУ из 31.07 закрываются 1–11.08
  AND x.cli_id IS NULL;                   -- нет закрытий не-ФУ в августе

CREATE UNIQUE CLUSTERED INDEX IX_elig_cli ON #eligible(cli_id);

/* 3) База: сколько клиентов и какой объём закрывался (по 31.07) */
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
SELECT
    seed_clients = COUNT(DISTINCT j.cli_id),
    vol_closed_31 = SUM(CASE WHEN j.is_fu=1 AND j.dt_close BETWEEN @CloseFrom AND @CloseTo THEN j.vol31 ELSE 0 END)
INTO #base
FROM #j31 j
JOIN #eligible e ON e.cli_id = j.cli_id;

/* 4) Снимок 12.08 для eligible: что открылось 01–12.08 и какой объём на 12.08 */
IF OBJECT_ID('tempdb..#a12') IS NOT NULL DROP TABLE #a12;
SELECT
    t.cli_id,
    is_fu   = CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END,
    dt_open = CAST(t.dt_open AS date),
    vol12   = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #a12
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #eligible e ON e.cli_id = t.cli_id
LEFT JOIN #fu  f ON f.prod_name_res = t.PROD_NAME_res
WHERE t.dt_rep       = @ChkDate
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  AND t.dt_open BETWEEN @CloseFrom AND @ChkDate
GROUP BY t.cli_id, CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END, t.dt_open;

CREATE CLUSTERED INDEX IX_a12_cli ON #a12(cli_id);

/* 5) Классификация клиентов по исходу + объёмы открытий на 12.08 */
WITH opened AS (
  SELECT
      cli_id,
      opened_fu    = MAX(CASE WHEN is_fu=1 THEN 1 ELSE 0 END),
      opened_bank  = MAX(CASE WHEN is_fu=0 THEN 1 ELSE 0 END),
      vol_open_fu  = SUM(CASE WHEN is_fu=1 THEN vol12 ELSE 0 END),
      vol_open_bank= SUM(CASE WHEN is_fu=0 THEN vol12 ELSE 0 END)
  FROM #a12
  GROUP BY cli_id
),
outcomes AS (
  SELECT
    o.cli_id,
    outcome = CASE
                WHEN o.opened_fu=1 AND o.opened_bank=0 THEN 'FU_only'
                WHEN o.opened_fu=0 AND o.opened_bank=1 THEN 'Bank_only'
                WHEN o.opened_fu=1 AND o.opened_bank=1 THEN 'FU_and_Bank'
                ELSE 'No_opens'
              END,
    o.vol_open_fu,
    o.vol_open_bank
  FROM opened o
),
summary_open AS (
  SELECT
    outcome,
    clients_cnt = COUNT(*),
    vol_fu   = SUM(vol_open_fu),
    vol_bank = SUM(vol_open_bank)
  FROM outcomes
  GROUP BY outcome
),
no_opens AS (
  SELECT
    clients_cnt = b.seed_clients - ISNULL( (SELECT SUM(clients_cnt) FROM summary_open WHERE outcome <> 'No_opens'), 0)
  FROM #base b
)
-- ===== Итоговый "принт" =====
SELECT 'SEED (31.07 только ФУ, все закрытия 1–11.08)' AS label,
       b.seed_clients      AS clients_cnt,
       b.vol_closed_31     AS volume_rub,
       NULL AS extra1, NULL AS extra2
FROM #base b

UNION ALL
SELECT 'OPEN: только Банк (01–12.08)', s.clients_cnt, s.vol_bank, NULL, NULL
FROM summary_open s WHERE s.outcome='Bank_only'

UNION ALL
SELECT 'OPEN: только ФУ (01–12.08)', s.clients_cnt, s.vol_fu, NULL, NULL
FROM summary_open s WHERE s.outcome='FU_only'

UNION ALL
SELECT 'OPEN: ФУ и Банк (01–12.08)', s.clients_cnt, s.vol_fu, s.vol_bank, NULL
FROM summary_open s WHERE s.outcome='FU_and_Bank'

UNION ALL
SELECT 'NO OPENS (ничего не открыто 01–12.08)', n.clients_cnt, CAST(0 AS decimal(20,2)), NULL, NULL
FROM no_opens n
ORDER BY CASE label
           WHEN 'SEED (31.07 только ФУ, все закрытия 1–11.08)' THEN 1
           WHEN 'OPEN: только Банк (01–12.08)' THEN 2
           WHEN 'OPEN: только ФУ (01–12.08)' THEN 3
           WHEN 'OPEN: ФУ и Банк (01–12.08)' THEN 4
           WHEN 'NO OPENS (ничего не открыто 01–12.08)' THEN 5
         END
OPTION (RECOMPILE, USE HINT('ENABLE_BATCH_MODE'));
