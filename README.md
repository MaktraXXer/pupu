SET NOCOUNT ON;

DECLARE @JulEOM      date = '2025-07-31';
DECLARE @ChkDate     date = '2025-08-12';
DECLARE @CloseFrom   date = '2025-08-01';  -- окно «закроется за дни августа»
DECLARE @CloseTo     date = '2025-08-31';  -- при желании поставь '2025-08-11'

/* 0) ФУ-справочник */
IF OBJECT_ID('tempdb..#fu') IS NOT NULL DROP TABLE #fu;
CREATE TABLE #fu (prod_name_res nvarchar(255) NOT NULL PRIMARY KEY);
INSERT INTO #fu(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* 1) Когорта: ФУ-закрытия (если нужно оставить как в твоём пайплайне) */
IF OBJECT_ID('tempdb..#cohort') IS NOT NULL DROP TABLE #cohort;
SELECT DISTINCT t.cli_id
INTO #cohort
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #fu f ON f.prod_name_res = t.PROD_NAME_res
WHERE t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  AND t.dt_close BETWEEN '2025-08-01' AND '2025-08-11';  -- как раньше (можешь убрать, если не нужно)
CREATE UNIQUE CLUSTERED INDEX IX_cohort_cli ON #cohort(cli_id);

/* 2) Seed: клиенты, у кого ФУ было на 31.07 (отрезаем OUT) */
IF OBJECT_ID('tempdb..#seed_0731') IS NOT NULL DROP TABLE #seed_0731;
SELECT DISTINCT t.cli_id
INTO #seed_0731
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
LEFT JOIN #cohort c ON c.cli_id = t.cli_id   -- если когорту не хочешь, замени на WHERE 1=1
JOIN #fu f         ON f.prod_name_res = t.PROD_NAME_res
WHERE t.dt_rep = @JulEOM
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL);
CREATE UNIQUE CLUSTERED INDEX IX_seed_cli ON #seed_0731(cli_id);

/* 3) Снимок 31.07 по договорам (нужны dt_close) */
IF OBJECT_ID('tempdb..#july31') IS NOT NULL DROP TABLE #july31;
SELECT
    t.con_id,
    t.cli_id,
    CAST(t.dt_open  AS date) AS dt_open,
    CAST(t.dt_close AS date) AS dt_close,
    vol31 = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #july31
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed_0731 s ON s.cli_id = t.cli_id
WHERE t.dt_rep = @JulEOM
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.con_id, t.cli_id, t.dt_open, t.dt_close;
CREATE UNIQUE CLUSTERED INDEX IX_july31_con ON #july31(con_id);

/* 4) Снимок 12.08 по договорам (нужны dt_open) */
IF OBJECT_ID('tempdb..#aug12') IS NOT NULL DROP TABLE #aug12;
SELECT
    t.con_id,
    t.cli_id,
    CAST(t.dt_open  AS date) AS dt_open,
    CAST(t.dt_close AS date) AS dt_close,
    vol12 = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #aug12
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed_0731 s ON s.cli_id = t.cli_id
WHERE t.dt_rep = @ChkDate
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.con_id, t.cli_id, t.dt_open, t.dt_close;
CREATE UNIQUE CLUSTERED INDEX IX_aug12_con ON #aug12(con_id);

/* 5) Разбивка объёмов */
WITH j AS (
  SELECT
    vol_31_close_aug     = SUM(CASE WHEN dt_close BETWEEN @CloseFrom AND @CloseTo THEN vol31 ELSE 0 END),
    vol_31_not_close_aug = SUM(CASE WHEN dt_close IS NULL OR dt_close < @CloseFrom OR dt_close > @CloseTo THEN vol31 ELSE 0 END)
  FROM #july31
),
a AS (
  SELECT
    vol_12_opened_aug_to_chk = SUM(CASE WHEN dt_open BETWEEN @CloseFrom AND @ChkDate THEN vol12 ELSE 0 END),
    vol_12_from_31_alive     = SUM(CASE WHEN j31.con_id IS NOT NULL THEN a12.vol12 ELSE 0 END)
  FROM #aug12 a12
  LEFT JOIN #july31 j31 ON j31.con_id = a12.con_id   -- «остался жить» = тот же договор был 31.07
)
SELECT
  /* 31.07 */
  j.vol_31_close_aug,
  j.vol_31_not_close_aug,
  total_31 = j.vol_31_close_aug + j.vol_31_not_close_aug,

  /* 12.08 */
  a.vol_12_opened_aug_to_chk,
  a.vol_12_from_31_alive,
  total_1208 = a.vol_12_opened_aug_to_chk + a.vol_12_from_31_alive
FROM j CROSS JOIN a
OPTION (RECOMPILE, USE HINT('ENABLE_BATCH_MODE'));****
