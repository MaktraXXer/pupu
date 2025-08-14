SET NOCOUNT ON;

DECLARE @JulEOM    date = '2025-07-31';
DECLARE @ChkDate   date = '2025-08-12';
DECLARE @CloseFrom date = '2025-08-01';
DECLARE @CloseTo   date = '2025-08-31';  -- если нужно только 01–11, поставь '2025-08-11'

/* 0) Справочник ФУ (крошечная #temp → дешёвый join) */
IF OBJECT_ID('tempdb..#fu') IS NOT NULL DROP TABLE #fu;
CREATE TABLE #fu (prod_name_res nvarchar(255) NOT NULL PRIMARY KEY);
INSERT INTO #fu(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* 1) Снимок 31.07 по договорам выбранных клиентов (нужны vol и dt_close) */
IF OBJECT_ID('tempdb..#j31') IS NOT NULL DROP TABLE #j31;
SELECT
    t.con_id,
    t.cli_id,
    is_fu   = CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END,
    dt_close= CAST(t.dt_close AS date),
    vol31   = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
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

/* 2) Оставляем ТОЛЬКО «чистых» клиентов:
      - на 31.07 у клиента нет НЕ-ФУ (только ФУ),
      - все его ФУ на 31.07 закрываются в августе,
      - в августе у него НЕТ закрытий по НЕ-ФУ вообще. */
IF OBJECT_ID('tempdb..#eligible') IS NOT NULL DROP TABLE #eligible;
WITH per_client AS (
  SELECT
      j.cli_id,
      has_nonfu_31       = MAX(CASE WHEN j.is_fu=0 THEN 1 ELSE 0 END),
      fu_cnt_31          = SUM(CASE WHEN j.is_fu=1 THEN 1 ELSE 0 END),
      fu_cnt_close_aug   = SUM(CASE WHEN j.is_fu=1 AND j.dt_close BETWEEN @CloseFrom AND @CloseTo THEN 1 ELSE 0 END)
  FROM #j31 j
  GROUP BY j.cli_id
),
nonfu_close_aug AS (   -- анти-условие: клиенты с не-ФУ закрытиями в августе (исключаем)
  SELECT DISTINCT t.cli_id
  FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
  LEFT JOIN #fu f ON f.prod_name_res = t.PROD_NAME_res
  WHERE CAST(t.dt_close AS date) BETWEEN @CloseFrom AND @CloseTo
    AND f.prod_name_res IS NULL   -- не-ФУ
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
  AND x.cli_id IS NULL;                   -- нет НЕ-ФУ закрытий в августе
CREATE UNIQUE CLUSTERED INDEX IX_elig_cli ON #eligible(cli_id);

/* 3) Объём «сколько закрылось» и «сколько клиентов закрылось» (для eligible) */
WITH closed31 AS (
  SELECT
      SUM(j.vol31)                    AS vol_closed_31aug,
      COUNT(DISTINCT j.cli_id)        AS clients_with_closures
  FROM #j31 j
  JOIN #eligible e ON e.cli_id = j.cli_id
  WHERE j.is_fu = 1
    AND j.dt_close BETWEEN @CloseFrom AND @CloseTo
)
SELECT * FROM closed31;

/* 4) Снимок 12.08 по договорам eligible (нужны dt_open и vol12) */
IF OBJECT_ID('tempdb..#a12') IS NOT NULL DROP TABLE #a12;
SELECT
    t.con_id,
    t.cli_id,
    is_fu  = CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END,
    dt_open= CAST(t.dt_open AS date),
    vol12  = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
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

/* 5) Открытия за 01–12.08 среди eligible:
      — общий объём/клиенты,
      — разложение на 3 исхода (FU-only / FU+Bank / Bank-only). */
;WITH opened AS (
  SELECT
      cli_id,
      opened_fu     = MAX(CASE WHEN is_fu=1 AND dt_open BETWEEN @CloseFrom AND @ChkDate THEN 1 ELSE 0 END),
      opened_nonfu  = MAX(CASE WHEN is_fu=0 AND dt_open BETWEEN @CloseFrom AND @ChkDate THEN 1 ELSE 0 END),
      vol_open_total= SUM(CASE WHEN dt_open BETWEEN @CloseFrom AND @ChkDate THEN vol12 ELSE 0 END),
      vol_open_fu   = SUM(CASE WHEN is_fu=1 AND dt_open BETWEEN @CloseFrom AND @ChkDate THEN vol12 ELSE 0 END),
      vol_open_bank = SUM(CASE WHEN is_fu=0 AND dt_open BETWEEN @CloseFrom AND @ChkDate THEN vol12 ELSE 0 END)
  FROM #a12
  GROUP BY cli_id
),
summary_total AS (
  SELECT
      clients_with_opens = COUNT(*) FILTER (WHERE (opened_fu=1 OR opened_nonfu=1)),
      vol_open_total     = SUM(vol_open_total)
  FROM opened
),
summary_by_outcome AS (
  SELECT outcome,
         clients_cnt = COUNT(*),
         vol_open    = SUM(CASE outcome
                              WHEN 'FU_only'      THEN vol_open_total
                              WHEN 'FU_and_Bank'  THEN vol_open_total
                              WHEN 'Bank_only'    THEN vol_open_total
                            END)
  FROM (
    SELECT *,
           CASE
             WHEN opened_fu=1 AND opened_nonfu=0 THEN 'FU_only'
             WHEN opened_fu=1 AND opened_nonfu=1 THEN 'FU_and_Bank'
             WHEN opened_fu=0 AND opened_nonfu=1 THEN 'Bank_only'
             ELSE 'No_opens'
           END AS outcome
    FROM opened
  ) x
  WHERE outcome <> 'No_opens'
  GROUP BY outcome
)
SELECT * FROM summary_total;           -- итого: у скольких открылись + общий объём открытий
SELECT * FROM summary_by_outcome;      -- разложение открытий: кто куда открылся
