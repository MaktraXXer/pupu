SET NOCOUNT ON;

DECLARE @JulEOM      date = '2025-07-31';
DECLARE @ChkDate     date = '2025-08-12';
DECLARE @CloseFrom   date = '2025-08-01';
DECLARE @CloseTo     date = '2025-08-11';   -- окно «1–11 августа»

/* 0) Справочник ФУ */
IF OBJECT_ID('tempdb..#fu') IS NOT NULL DROP TABLE #fu;
CREATE TABLE #fu (prod_name_res nvarchar(255) NOT NULL PRIMARY KEY);
INSERT INTO #fu(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* 1) Когорта: клиенты с ФУ-закрытиями 1–11 августа */
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
  AND t.dt_close BETWEEN @CloseFrom AND @CloseTo;
CREATE UNIQUE CLUSTERED INDEX IX_cohort_cli ON #cohort(cli_id);

/* 2) Seed: берём только клиентов, у кого ФУ было на 31.07 (отрезаем OUT) */
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
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL);
CREATE UNIQUE CLUSTERED INDEX IX_seed_cli ON #seed_0731(cli_id);

/* 3) Снимок 31.07 по договорам (для разбиения по закрытиям) */
IF OBJECT_ID('tempdb..#july31') IS NOT NULL DROP TABLE #july31;
SELECT
    t.con_id,
    t.cli_id,
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
GROUP BY t.con_id, t.cli_id, t.dt_close;
CREATE UNIQUE CLUSTERED INDEX IX_july31_con ON #july31(con_id);

/* 4) Снимок 12.08 по договорам (для разбиения по открытиям) */
IF OBJECT_ID('tempdb..#aug12') IS NOT NULL DROP TABLE #aug12;
SELECT
    t.con_id,
    t.cli_id,
    CAST(t.dt_open  AS date) AS dt_open,
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
GROUP BY t.con_id, t.cli_id, t.dt_open;
CREATE UNIQUE CLUSTERED INDEX IX_aug12_con ON #aug12(con_id);

/* 5) Флаги состояний и total-объёмы по клиенту на обе даты (для A0/B0 → A1/B1/C1/N1) */
IF OBJECT_ID('tempdb..#state') IS NOT NULL DROP TABLE #state;
WITH s_raw AS (
  SELECT t.cli_id,
         vol_total_0731 = SUM(CASE WHEN t.dt_rep=@JulEOM  AND t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END),
         vol_total_0812 = SUM(CASE WHEN t.dt_rep=@ChkDate AND t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END),
         has_fu_0731    = MAX(CASE WHEN t.dt_rep=@JulEOM  AND f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END),
         has_nonfu_0731 = MAX(CASE WHEN t.dt_rep=@JulEOM  AND f.prod_name_res IS NULL THEN 1 ELSE 0 END),
         has_fu_0812    = MAX(CASE WHEN t.dt_rep=@ChkDate AND f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END),
         has_nonfu_0812 = MAX(CASE WHEN t.dt_rep=@ChkDate AND f.prod_name_res IS NULL THEN 1 ELSE 0 END)
  FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
  JOIN #seed_0731 s ON s.cli_id = t.cli_id
  LEFT JOIN #fu f   ON f.prod_name_res = t.PROD_NAME_res
  WHERE t.dt_rep IN (@JulEOM, @ChkDate)
    AND t.block_name=N'Привлечение ФЛ' AND t.od_flag=1 AND t.cur='810'
    AND t.out_rub IS NOT NULL
    AND t.section_name IN (N'Срочные',N'Срочные ')
    AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  GROUP BY t.cli_id
)
SELECT
  cli_id,
  vol_total_0731,
  vol_total_0812,
  has_fu_0731,
  has_nonfu_0731,
  has_fu_0812,
  has_nonfu_0812
INTO #state
FROM s_raw;
CREATE UNIQUE CLUSTERED INDEX IX_state_cli ON #state(cli_id);

/* 6) Классификация путей по клиенту */
IF OBJECT_ID('tempdb..#class') IS NOT NULL DROP TABLE #class;
SELECT
  st.cli_id,
  st.vol_total_0731,
  st.vol_total_0812,
  initial_state = CASE WHEN st.has_fu_0731=1 AND st.has_nonfu_0731=0 THEN 'A0' ELSE 'B0' END,
  final_state   = CASE
                    WHEN st.has_fu_0812=1 AND st.has_nonfu_0812=0 THEN 'A1'
                    WHEN st.has_fu_0812=1 AND st.has_nonfu_0812=1 THEN 'B1'
                    WHEN st.has_fu_0812=0 AND st.has_nonfu_0812=1 THEN 'C1'
                    ELSE 'N1'
                  END
INTO #class
FROM #state st;
CREATE UNIQUE CLUSTERED INDEX IX_class_cli ON #class(cli_id);

/* 7) Разбивка объёмов по клиенту на 31.07 и 12.08 */
IF OBJECT_ID('tempdb..#b31') IS NOT NULL DROP TABLE #b31;
SELECT
  j.cli_id,
  vol_31_total        = SUM(j.vol31),
  vol_31_close_plan   = SUM(CASE WHEN j.dt_close BETWEEN @CloseFrom AND @CloseTo THEN j.vol31 ELSE 0 END),
  vol_31_other        = SUM(j.vol31) - SUM(CASE WHEN j.dt_close BETWEEN @CloseFrom AND @CloseTo THEN j.vol31 ELSE 0 END)
INTO #b31
FROM #july31 j
GROUP BY j.cli_id;
CREATE UNIQUE CLUSTERED INDEX IX_b31_cli ON #b31(cli_id);

IF OBJECT_ID('tempdb..#b12') IS NOT NULL DROP TABLE #b12;
SELECT
  a.cli_id,
  vol_12_total      = SUM(a.vol12),
  vol_12_open_aug   = SUM(CASE WHEN a.dt_open BETWEEN @CloseFrom AND @CloseTo THEN a.vol12 ELSE 0 END),
  vol_12_other      = SUM(a.vol12) - SUM(CASE WHEN a.dt_open BETWEEN @CloseFrom AND @CloseTo THEN a.vol12 ELSE 0 END)
INTO #b12
FROM #aug12 a
GROUP BY a.cli_id;
CREATE UNIQUE CLUSTERED INDEX IX_b12_cli ON #b12(cli_id);

/* 8) ФИНАЛ — тот же формат, но с разбивкой объёмов на 2+2 колонки */
SELECT
  c.initial_state,             -- 'A0' / 'B0'
  c.final_state,               -- 'A1' / 'B1' / 'C1' / 'N1'

  /* объёмы на 31.07 (2 колонки вместо vol_total_start) */
  vol_start_close_plan = SUM(ISNULL(b31.vol_31_close_plan,0)),   -- был 31.07 и закроется 1–11.08
  vol_start_other      = SUM(ISNULL(b31.vol_31_other,0)),        -- остальной объём на 31.07

  /* объёмы на 12.08 (2 колонки вместо vol_total_end) */
  vol_end_open_aug     = SUM(ISNULL(b12.vol_12_open_aug,0)),     -- открылся 1–11.08
  vol_end_other        = SUM(ISNULL(b12.vol_12_other,0)),        -- остальной объём на 12.08

  /* количество клиентов — как было */
  path_clients_cnt = COUNT(*),
  clients_start    = COUNT(DISTINCT CASE WHEN c.vol_total_0731 > 0 THEN c.cli_id END),
  clients_end      = COUNT(DISTINCT CASE WHEN c.vol_total_0812 > 0 THEN c.cli_id END)

FROM #class c
LEFT JOIN #b31 b31 ON b31.cli_id = c.cli_id
LEFT JOIN #b12 b12 ON b12.cli_id = c.cli_id
GROUP BY c.initial_state, c.final_state
ORDER BY c.initial_state, c.final_state
OPTION (RECOMPILE, USE HINT('ENABLE_BATCH_MODE'));
