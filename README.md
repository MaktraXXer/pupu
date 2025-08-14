SET NOCOUNT ON;

DECLARE @JulEOM    date = '2025-07-31';
DECLARE @ChkDate   date = '2025-08-12';
DECLARE @CloseFrom date = '2025-08-01';
DECLARE @CloseTo   date = '2025-08-31';   -- если нужно 01–11, поставь '2025-08-11'

/* 0) Справочник ФУ */
IF OBJECT_ID('tempdb..#fu') IS NOT NULL DROP TABLE #fu;
CREATE TABLE #fu (prod_name_res nvarchar(255) NOT NULL PRIMARY KEY);
INSERT INTO #fu(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* 1) Снимок 31.07 по договорам */
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
CREATE UNIQUE CLUSTERED INDEX IX_j31_con ON #j31(con_id);

/* 2) Оставляем «чистых» клиентов:
      — на 31.07 нет не-ФУ (только ФУ),
      — все их ФУ на 31.07 закрываются в августе,
      — в августе нет закрытий по не-ФУ. */
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
nonfu_close_aug AS (   -- клиенты с не-ФУ закрытиями в августе (исключаем)
  SELECT DISTINCT t.cli_id
  FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
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
WHERE pc.has_nonfu_31 = 0                 -- только ФУ на 31.07
  AND pc.fu_cnt_31 > 0
  AND pc.fu_cnt_close_aug = pc.fu_cnt_31  -- все ФУ закрываются в августе
  AND x.cli_id IS NULL;                   -- нет закрытий не-ФУ в августе
CREATE UNIQUE CLUSTERED INDEX IX_elig_cli ON #eligible(cli_id);

/* 3) «Сколько закрылось» (объём и клиенты) среди eligible */
SELECT
    vol_closed_31aug = SUM(CASE WHEN j.dt_close BETWEEN @CloseFrom AND @CloseTo THEN j.vol31 ELSE 0 END),
    clients_with_closures = COUNT(DISTINCT CASE WHEN j.dt_close BETWEEN @CloseFrom AND @CloseTo THEN j.cli_id END)
FROM #j31 j
JOIN #eligible e ON e.cli_id = j.cli_id
WHERE j.is_fu = 1;

/* 4) Снимок 12.08 по договорам eligible */
IF OBJECT_ID('tempdb..#a12') IS NOT NULL DROP TABLE #a12;
SELECT
    t.con_id,
    t.cli_id,
    is_fu   = CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END,
    dt_open = CAST(t.dt_open AS date),
    vol12   = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #a12
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #eligible e ON e.cli_id = t.cli_id
LEFT JOIN #fu f  ON f.prod_name_res = t.PROD_NAME_res
WHERE t.dt_rep       = @ChkDate
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.con_id, t.cli_id, CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END, t.dt_open;
CREATE UNIQUE CLUSTERED INDEX IX_a12_con ON #a12(con_id);

/* 5) «Сколько открылось» (объём и клиенты) + разложение по исходам */
WITH opened AS (
  SELECT
      cli_id,
      opened_fu    = MAX(CASE WHEN is_fu=1 AND dt_open BETWEEN @CloseFrom AND @ChkDate THEN 1 ELSE 0 END),
      opened_bank  = MAX(CASE WHEN is_fu=0 AND dt_open BETWEEN @CloseFrom AND @ChkDate THEN 1 ELSE 0 END),
      vol_open_fu  = SUM(CASE WHEN is_fu=1 AND dt_open BETWEEN @CloseFrom AND @ChkDate THEN vol12 ELSE 0 END),
      vol_open_bank= SUM(CASE WHEN is_fu=0 AND dt_open BETWEEN @CloseFrom AND @ChkDate THEN vol12 ELSE 0 END)
  FROM #a12
  GROUP BY cli_id
),
summary_total AS (
  SELECT
      clients_with_opens = SUM(CASE WHEN (opened_fu=1 OR opened_bank=1) THEN 1 ELSE 0 END),
      vol_open_total     = SUM(vol_open_fu + vol_open_bank)
  FROM opened
),
by_outcome AS (
  SELECT
      outcome,
      clients_cnt = COUNT(*) ,
      vol_open_total = SUM(vol_open_fu + vol_open_bank),
      vol_open_fu    = SUM(vol_open_fu),
      vol_open_bank  = SUM(vol_open_bank)
  FROM (
      SELECT *,
             CASE
               WHEN opened_fu=1 AND opened_bank=0 THEN 'FU_only'
               WHEN opened_fu=1 AND opened_bank=1 THEN 'FU_and_Bank'
               WHEN opened_fu=0 AND opened_bank=1 THEN 'Bank_only'
               ELSE 'No_opens'
             END AS outcome
      FROM opened
  ) x
  WHERE outcome <> 'No_opens'
  GROUP BY outcome
)
SELECT * FROM summary_total;   -- у скольких открылось и какой общий объём открытий
SELECT * FROM by_outcome;      -- разложение открытий: FU_only / FU_and_Bank / Bank_only
