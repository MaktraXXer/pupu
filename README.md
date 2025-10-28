/* ===================== ПАРАМЕТРЫ ===================== */
SET NOCOUNT ON;
DECLARE @cur varchar(3)='810', @acc_role nvarchar(10)=N'LIAB', @od_only bit=1;

DECLARE @dt_rep_prev date='2025-09-30';  -- НС "до"
DECLARE @dt_rep_now  date='2025-10-25';  -- НС "после" + открытия

DECLARE @w1_from date='2025-10-01', @w1_to date='2025-10-15';
DECLARE @w2_from date='2025-10-16', @w2_to date='2025-10-25';

DECLARE @section_td nvarchar(50)=N'Срочные';
DECLARE @block_fl   nvarchar(100)=N'Привлечение ФЛ';
DECLARE @section_ns nvarchar(50)=N'Накопительный счёт';

DECLARE @eps decimal(9,6)=0.0005, @rk_split_from date='2025-10-16';
DECLARE @month_start date = DATEFROMPARTS(YEAR(@dt_rep_now), MONTH(@dt_rep_now), 1);

DECLARE @cohort_mode varchar(16)='W1_ONLY';  -- 'W1_ONLY' | 'W2_ONLY' | 'BOTH'

/* ================== МАРКЕТ-исключения ================= */
IF OBJECT_ID('tempdb..#market_names') IS NOT NULL DROP TABLE #market_names;
CREATE TABLE #market_names(prod_name_res nvarchar(200) PRIMARY KEY);
INSERT INTO #market_names VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),(N'Надёжный T2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный'),
 (N'ДОМа надёжно'),(N'Всё в ДОМ');

/* ================== РК-ставки по периодам ============== */
IF OBJECT_ID('tempdb..#rk_rates') IS NOT NULL DROP TABLE #rk_rates;
CREATE TABLE #rk_rates(
  d_from date, d_to date, conv_type varchar(20), r decimal(9,6),
  INDEX IX_rk (conv_type, d_from, d_to)
);
INSERT INTO #rk_rates VALUES
(@month_start, DATEADD(day,-1,@rk_split_from),'AT_THE_END',0.165),
(@month_start, DATEADD(day,-1,@rk_split_from),'AT_THE_END',0.163),
(@month_start, DATEADD(day,-1,@rk_split_from),'NOT_AT_THE_END',0.162),
(@month_start, DATEADD(day,-1,@rk_split_from),'NOT_AT_THE_END',0.160),
(@rk_split_from,@dt_rep_now,'AT_THE_END',0.170),
(@rk_split_from,@dt_rep_now,'AT_THE_END',0.168),
(@rk_split_from,@dt_rep_now,'NOT_AT_THE_END',0.167),
(@rk_split_from,@dt_rep_now,'NOT_AT_THE_END',0.165);

/* ============ 30.09: ВЫХОДЫ (срочные, НЕ-маркет) ======== */
IF OBJECT_ID('tempdb..#exits_by_cli') IS NOT NULL DROP TABLE #exits_by_cli;
CREATE TABLE #exits_by_cli(
  cli_id bigint PRIMARY KEY,
  dep_sum_w1 decimal(20,4) NOT NULL,
  dep_sum_w2 decimal(20,4) NOT NULL
);
INSERT INTO #exits_by_cli(cli_id, dep_sum_w1, dep_sum_w2)
SELECT
  t.cli_id,
  SUM(CASE WHEN TRY_CAST(t.dt_close AS date) BETWEEN @w1_from AND @w1_to THEN t.out_rub ELSE 0 END) AS dep_sum_w1,
  SUM(CASE WHEN TRY_CAST(t.dt_close AS date) BETWEEN @w2_from AND @w2_to THEN t.out_rub ELSE 0 END) AS dep_sum_w2
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
LEFT JOIN #market_names m ON m.prod_name_res = t.prod_name_res
WHERE
  t.dt_rep=@dt_rep_prev AND t.section_name=@section_td AND t.block_name=@block_fl
  AND (@od_only=0 OR t.od_flag=1) AND t.cur=@cur AND t.acc_role=@acc_role
  AND t.out_rub IS NOT NULL AND t.out_rub>=0 AND m.prod_name_res IS NULL
GROUP BY t.cli_id
HAVING SUM(CASE WHEN TRY_CAST(t.dt_close AS date) BETWEEN @w1_from AND @w1_to THEN t.out_rub ELSE 0 END) > 0
    OR SUM(CASE WHEN TRY_CAST(t.dt_close AS date) BETWEEN @w2_from AND @w2_to THEN t.out_rub ELSE 0 END) > 0;

/* ====== COHORT: выбор сценария клиентов (W1_ONLY/W2_ONLY/BOTH) ====== */
IF OBJECT_ID('tempdb..#cohort') IS NOT NULL DROP TABLE #cohort;
CREATE TABLE #cohort(cli_id bigint PRIMARY KEY);
IF @cohort_mode='W1_ONLY'
  INSERT INTO #cohort SELECT cli_id FROM #exits_by_cli WHERE dep_sum_w1>0 AND dep_sum_w2=0;
ELSE IF @cohort_mode='W2_ONLY'
  INSERT INTO #cohort SELECT cli_id FROM #exits_by_cli WHERE dep_sum_w2>0 AND dep_sum_w1=0;
ELSE IF @cohort_mode='BOTH'
  INSERT INTO #cohort SELECT cli_id FROM #exits_by_cli WHERE dep_sum_w1>0 AND dep_sum_w2>0;
ELSE
  BEGIN RAISERROR('Unknown @cohort_mode. Use W1_ONLY | W2_ONLY | BOTH',16,1); RETURN; END;

-- Сверка: корректный clients_cnt + раздельные суммы выходов
SELECT
  @cohort_mode AS cohort_mode,
  (SELECT COUNT(*) FROM #cohort) AS clients_cnt,
  ISNULL(SUM(e.dep_sum_w1),0) AS deposits_maturing_sum_w1,
  ISNULL(SUM(e.dep_sum_w2),0) AS deposits_maturing_sum_w2
FROM #exits_by_cli e
JOIN #cohort c ON c.cli_id=e.cli_id;

 /* ============ НС 30.09 и 25.10 только по пулу ============ */
-- НС 30.09
SELECT
  COUNT(*) AS clients_ns_0930_cnt,
  SUM(n.out_rub) AS ns_0930_sum
FROM (
  SELECT t.cli_id, SUM(t.out_rub) AS out_rub
  FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
  JOIN #cohort c ON c.cli_id=t.cli_id
  WHERE t.dt_rep=@dt_rep_prev AND t.section_name=@section_ns AND t.block_name=@block_fl
    AND (@od_only=0 OR t.od_flag=1) AND t.cur=@cur
    AND t.out_rub IS NOT NULL AND t.out_rub>=0
  GROUP BY t.cli_id
) n;

-- НС 25.10
SELECT
  COUNT(*) AS clients_ns_1025_cnt,
  SUM(n.out_rub) AS ns_1025_sum
FROM (
  SELECT t.cli_id, SUM(t.out_rub) AS out_rub
  FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
  JOIN #cohort c ON c.cli_id=t.cli_id
  WHERE t.dt_rep=@dt_rep_now AND t.section_name=@section_ns AND t.block_name=@block_fl
    AND (@od_only=0 OR t.od_flag=1) AND t.cur=@cur
    AND t.out_rub IS NOT NULL AND t.out_rub>=0
  GROUP BY t.cli_id
) n;

/* ======== 25.10: ОТКРЫТИЯ по пулу (НЕ-маркет) с флагом РК ======== */
IF OBJECT_ID('tempdb..#open_raw') IS NOT NULL DROP TABLE #open_raw;
CREATE TABLE #open_raw(
  dt_open_d date, cli_id bigint, out_rub decimal(20,4), rate_con decimal(9,6), conv_norm varchar(20),
  INDEX IX_open (dt_open_d, cli_id)
);
INSERT INTO #open_raw(dt_open_d, cli_id, out_rub, rate_con, conv_norm)
SELECT
  CAST(t.dt_open AS date),
  t.cli_id,
  t.out_rub,
  t.rate_con,
  CASE
    WHEN NULLIF(LTRIM(RTRIM(COALESCE(t.conv,''))), '') IS NULL THEN 'AT_THE_END'
    ELSE UPPER(LTRIM(RTRIM(t.conv)))
  END
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
JOIN #cohort c ON c.cli_id=t.cli_id
LEFT JOIN #market_names m ON m.prod_name_res=t.prod_name_res
WHERE
  t.dt_rep=@dt_rep_now AND t.section_name=@section_td AND t.block_name=@block_fl
  AND (@od_only=0 OR t.od_flag=1) AND t.cur=@cur AND t.acc_role=@acc_role
  AND t.out_rub IS NOT NULL AND t.out_rub>=0
  AND m.prod_name_res IS NULL
  AND CAST(t.dt_open AS date) BETWEEN @w1_from AND @w2_to;

-- Флаг РК
IF OBJECT_ID('tempdb..#open_flag') IS NOT NULL DROP TABLE #open_flag;
CREATE TABLE #open_flag(
  window_label nvarchar(16), is_rk bit, cli_id bigint, out_rub decimal(20,4)
);
INSERT INTO #open_flag(window_label,is_rk,cli_id,out_rub)
SELECT
  CASE WHEN o.dt_open_d BETWEEN @w1_from AND @w1_to THEN N'01-15 Oct' ELSE N'16-25 Oct' END AS window_label,
  CASE
    WHEN o.conv_norm='AT_THE_END'
         AND EXISTS (SELECT 1 FROM #rk_rates rr WHERE rr.conv_type='AT_THE_END'
                     AND o.dt_open_d BETWEEN rr.d_from AND rr.d_to
                     AND ABS(o.rate_con - rr.r) <= @eps) THEN 1
    WHEN o.conv_norm<>'AT_THE_END'
         AND EXISTS (SELECT 1 FROM #rk_rates rr WHERE rr.conv_type='NOT_AT_THE_END'
                     AND o.dt_open_d BETWEEN rr.d_from AND rr.d_to
                     AND ABS(o.rate_con - rr.r) <= @eps) THEN 1
    ELSE 0
  END AS is_rk,
  o.cli_id,
  o.out_rub
FROM #open_raw o;

/* ======== Итог по открытиям: сумма + количество клиентов в категории ======== */
-- 01–15 по РК
SELECT N'01-15 Oct' AS window_label, N'По РК' AS rk_bucket,
       COUNT(DISTINCT cli_id) AS clients_cnt, SUM(out_rub) AS deposits_opened_sum
FROM #open_flag WHERE window_label=N'01-15 Oct' AND is_rk=1;

-- 01–15 не по РК
SELECT N'01-15 Oct' AS window_label, N'Не по РК' AS rk_bucket,
       COUNT(DISTINCT cli_id) AS clients_cnt, SUM(out_rub) AS deposits_opened_sum
FROM #open_flag WHERE window_label=N'01-15 Oct' AND is_rk=0;

-- 16–25 по РК
SELECT N'16-25 Oct' AS window_label, N'По РК' AS rk_bucket,
       COUNT(DISTINCT cli_id) AS clients_cnt, SUM(out_rub) AS deposits_opened_sum
FROM #open_flag WHERE window_label=N'16-25 Oct' AND is_rk=1;

-- 16–25 не по РК
SELECT N'16-25 Oct' AS window_label, N'Не по РК' AS rk_bucket,
       COUNT(DISTINCT cli_id) AS clients_cnt, SUM(out_rub) AS deposits_opened_sum
FROM #open_flag WHERE window_label=N'16-25 Oct' AND is_rk=0;
