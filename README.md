SET NOCOUNT ON;

DECLARE @JulEOM    date = '2025-07-31';
DECLARE @ChkDate   date = '2025-08-12';
DECLARE @CloseFrom date = '2025-08-01';
DECLARE @CloseTo   date = '2025-08-11';

/* 0) Крошечный справочник ФУ */
IF OBJECT_ID('tempdb..#fu') IS NOT NULL DROP TABLE #fu;
CREATE TABLE #fu (prod_name_res nvarchar(255) NOT NULL PRIMARY KEY);
INSERT INTO #fu(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* 1) Когорта: клиенты с ФУ-закрытиями в окне 01–11.08 (только базовая таблица) */
IF OBJECT_ID('tempdb..#cohort') IS NOT NULL DROP TABLE #cohort;
SELECT DISTINCT t.cli_id
INTO #cohort
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #fu f ON f.prod_name_res = t.PROD_NAME_res
WHERE t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные',N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  AND t.dt_close BETWEEN @CloseFrom AND @CloseTo;

CREATE UNIQUE CLUSTERED INDEX IX_cohort_cli ON #cohort(cli_id);

/* 2) Seed: из когорты оставляем только тех, у кого ФУ на 31.07 (режем OUT сразу) */
IF OBJECT_ID('tempdb..#seed_0731') IS NOT NULL DROP TABLE #seed_0731;
SELECT DISTINCT t.cli_id
INTO #seed_0731
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #cohort c ON c.cli_id = t.cli_id
JOIN #fu f     ON f.prod_name_res = t.PROD_NAME_res
WHERE t.dt_rep = @JulEOM
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные',N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL);

CREATE UNIQUE CLUSTERED INDEX IX_seed_cli ON #seed_0731(cli_id);

/* 3) ДВА SEEK-а по dt_rep, агрегации по всем вкладам и флагам ФУ/не-ФУ */
WITH agg AS (
  /* 3.1) 31.07 */
  SELECT
      t.cli_id,
      vol_total_0731 = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END),
      has_fu_0731    = 1,  -- гарантированно (по #seed)
      has_nonfu_0731 = MAX(CASE WHEN f.prod_name_res IS NULL THEN 1 ELSE 0 END),
      vol_total_0812 = CAST(0 AS decimal(20,2)),
      has_fu_0812    = 0,
      has_nonfu_0812 = 0
  FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
  JOIN #seed_0731 s ON s.cli_id = t.cli_id
  LEFT JOIN #fu f   ON f.prod_name_res = t.PROD_NAME_res
  WHERE t.dt_rep = @JulEOM
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag      = 1
    AND t.cur          = '810'
    AND t.out_rub     IS NOT NULL
    AND t.section_name IN (N'Срочные',N'Срочные ')
    AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  GROUP BY t.cli_id

  UNION ALL

  /* 3.2) 12.08 */
  SELECT
      t.cli_id,
      vol_total_0731 = CAST(0 AS decimal(20,2)),
      has_fu_0731    = 0,
      has_nonfu_0731 = 0,
      vol_total_0812 = SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END),
      has_fu_0812    = MAX(CASE WHEN f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END),
      has_nonfu_0812 = MAX(CASE WHEN f.prod_name_res IS NULL THEN 1 ELSE 0 END)
  FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
  JOIN #seed_0731 s ON s.cli_id = t.cli_id
  LEFT JOIN #fu f   ON f.prod_name_res = t.PROD_NAME_res
  WHERE t.dt_rep = @ChkDate
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag      = 1
    AND t.cur          = '810'
    AND t.out_rub     IS NOT NULL
    AND t.section_name IN (N'Срочные',N'Срочные ')
    AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  GROUP BY t.cli_id
),
agg2 AS (
  SELECT
    cli_id,
    vol_total_0731 = SUM(vol_total_0731),
    vol_total_0812 = SUM(vol_total_0812),
    has_fu_0731    = MAX(has_fu_0731),
    has_nonfu_0731 = MAX(has_nonfu_0731),
    has_fu_0812    = MAX(has_fu_0812),
    has_nonfu_0812 = MAX(has_nonfu_0812)
  FROM agg
  GROUP BY cli_id
),
classified AS (
  SELECT
    cli_id,
    vol_total_0731, vol_total_0812,
    initial_state = CASE WHEN has_nonfu_0731=0 THEN 'A0' ELSE 'B0' END,
    final_state   = CASE
                      WHEN has_fu_0812=1 AND has_nonfu_0812=0 THEN 'A1'
                      WHEN has_fu_0812=1 AND has_nonfu_0812=1 THEN 'B1'
                      WHEN has_fu_0812=0 AND has_nonfu_0812=1 THEN 'C1'
                      ELSE 'N1'
                    END
  FROM agg2
)
SELECT
  initial_state,                    -- 'A0' / 'B0'
  final_state,                      -- 'A1' / 'B1' / 'C1' / 'N1'
  SUM(vol_total_0731) AS vol_total_start,   -- объём на 31.07 (все вклады)
  COUNT(*)            AS clients_start,     -- та же группа клиентов
  COUNT(*)            AS clients_end,
  SUM(vol_total_0812) AS vol_total_end      -- объём на 12.08 (все вклады)
FROM classified
GROUP BY initial_state, final_state
ORDER BY initial_state, final_state
OPTION (RECOMPILE, USE HINT('ENABLE_BATCH_MODE'));  -- SQL Server 2019+: batch mode на rowstore
