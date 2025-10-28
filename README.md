/* ============================================================
   ПАРАМЕТРЫ
   ============================================================*/
SET NOCOUNT ON;

DECLARE @cur            varchar(3)    = '810';
DECLARE @acc_role       nvarchar(10)  = N'LIAB';
DECLARE @od_only        bit           = 1;

DECLARE @dt_rep_prev    date          = '2025-09-30';  -- снимок "до" (НС на 30.09)
DECLARE @dt_rep_now     date          = '2025-10-25';  -- снимок "после" (НС на 25.10 и открытия)

-- Декады (окна для выходов и открытий)
DECLARE @w1_from        date          = '2025-10-01';
DECLARE @w1_to          date          = '2025-10-15';
DECLARE @w2_from        date          = '2025-10-16';
DECLARE @w2_to          date          = '2025-10-25';

-- Выбор сценария клиентского пула: 'W1_ONLY' | 'W2_ONLY' | 'BOTH'
DECLARE @cohort_mode    varchar(16)   = 'W1_ONLY';

-- Фильтры секций
DECLARE @section_td     nvarchar(50)  = N'Срочные';
DECLARE @block_fl       nvarchar(100) = N'Привлечение ФЛ';
DECLARE @section_ns     nvarchar(50)  = N'Накопительный счёт';

-- Параметры «РК»
DECLARE @eps            decimal(9,6)  = 0.0005;
DECLARE @rk_split_from  date          = '2025-10-16';
DECLARE @month_start    date          = DATEFROMPARTS(YEAR(@dt_rep_now), MONTH(@dt_rep_now), 1);

/* ============================================================
   СПРАВОЧНИК: ИСКЛЮЧАЕМЫЕ ПРОДУКТЫ («МАРКЕТЫ»)
   ============================================================*/
IF OBJECT_ID('tempdb..#market_names') IS NOT NULL DROP TABLE #market_names;
CREATE TABLE #market_names(prod_name_res nvarchar(200) PRIMARY KEY);
INSERT INTO #market_names(prod_name_res) VALUES
 (N'Надёжный прайм'), (N'Надёжный VIP'), (N'Надёжный премиум'),
 (N'Надёжный промо'), (N'Надёжный старт'),
 (N'Надёжный Т2'), (N'Надёжный T2'),
 (N'Надёжный Мегафон'), (N'Надёжный процент'),
 (N'Надёжный'), (N'ДОМа надёжно'), (N'Всё в ДОМ');

/* ============================================================
   СПРАВОЧНИК РК-СТАВОК ПО ПЕРИОДАМ
   ============================================================*/
IF OBJECT_ID('tempdb..#rk_rates') IS NOT NULL DROP TABLE #rk_rates;
CREATE TABLE #rk_rates(
    d_from    date         NOT NULL,
    d_to      date         NOT NULL,
    conv_type varchar(20)  NOT NULL,   -- 'AT_THE_END' | 'NOT_AT_THE_END'
    r         decimal(9,6) NOT NULL
);
INSERT INTO #rk_rates(d_from,d_to,conv_type,r)
VALUES
(@month_start, DATEADD(day,-1,@rk_split_from), 'AT_THE_END',     0.165),
(@month_start, DATEADD(day,-1,@rk_split_from), 'AT_THE_END',     0.163),
(@month_start, DATEADD(day,-1,@rk_split_from), 'NOT_AT_THE_END', 0.162),
(@month_start, DATEADD(day,-1,@rk_split_from), 'NOT_AT_THE_END', 0.160),
(@rk_split_from, @dt_rep_now, 'AT_THE_END',     0.170),
(@rk_split_from, @dt_rep_now, 'AT_THE_END',     0.168),
(@rk_split_from, @dt_rep_now, 'NOT_AT_THE_END', 0.167),
(@rk_split_from, @dt_rep_now, 'NOT_AT_THE_END', 0.165);

/* ============================================================
   A) ВЫХОДЫ НА 30.09: формируем базы по декадам (НЕ-МАРКЕТ)
   ============================================================*/
IF OBJECT_ID('tempdb..#td_0930') IS NOT NULL DROP TABLE #td_0930;
CREATE TABLE #td_0930(
    cli_id   bigint      NOT NULL,
    out_rub  decimal(20,4) NOT NULL,
    dt_close date NULL
);

INSERT INTO #td_0930(cli_id, out_rub, dt_close)
SELECT
    t.cli_id,
    t.out_rub,
    TRY_CAST(t.dt_close AS date)
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
LEFT JOIN #market_names m ON m.prod_name_res = t.prod_name_res
WHERE
    t.dt_rep        = @dt_rep_prev
    AND t.section_name = @section_td
    AND t.block_name   = @block_fl
    AND (@od_only = 0 OR t.od_flag = 1)
    AND t.cur          = @cur
    AND t.acc_role     = @acc_role
    AND t.out_rub IS NOT NULL AND t.out_rub >= 0
    AND m.prod_name_res IS NULL;

IF OBJECT_ID('tempdb..#mature_w1_by_cli') IS NOT NULL DROP TABLE #mature_w1_by_cli;
CREATE TABLE #mature_w1_by_cli(cli_id bigint PRIMARY KEY, dep_sum_w1 decimal(20,4));
INSERT INTO #mature_w1_by_cli(cli_id, dep_sum_w1)
SELECT cli_id, SUM(out_rub)
FROM #td_0930
WHERE dt_close BETWEEN @w1_from AND @w1_to
GROUP BY cli_id;

IF OBJECT_ID('tempdb..#mature_w2_by_cli') IS NOT NULL DROP TABLE #mature_w2_by_cli;
CREATE TABLE #mature_w2_by_cli(cli_id bigint PRIMARY KEY, dep_sum_w2 decimal(20,4));
INSERT INTO #mature_w2_by_cli(cli_id, dep_sum_w2)
SELECT cli_id, SUM(out_rub)
FROM #td_0930
WHERE dt_close BETWEEN @w2_from AND @w2_to
GROUP BY cli_id;

/* ============================================================
   B) СЦЕНАРНЫЙ ПУЛ КЛИЕНТОВ: W1_ONLY / W2_ONLY / BOTH
   ============================================================*/
IF OBJECT_ID('tempdb..#clients_cohort') IS NOT NULL DROP TABLE #clients_cohort;
CREATE TABLE #clients_cohort(cli_id bigint PRIMARY KEY);

IF @cohort_mode = 'W1_ONLY'
BEGIN
    INSERT INTO #clients_cohort(cli_id)
    SELECT w1.cli_id
    FROM #mature_w1_by_cli w1
    WHERE NOT EXISTS (SELECT 1 FROM #mature_w2_by_cli w2 WHERE w2.cli_id = w1.cli_id);
END
ELSE IF @cohort_mode = 'W2_ONLY'
BEGIN
    INSERT INTO #clients_cohort(cli_id)
    SELECT w2.cli_id
    FROM #mature_w2_by_cli w2
    WHERE NOT EXISTS (SELECT 1 FROM #mature_w1_by_cli w1 WHERE w1.cli_id = w2.cli_id);
END
ELSE IF @cohort_mode = 'BOTH'
BEGIN
    INSERT INTO #clients_cohort(cli_id)
    SELECT w1.cli_id
    FROM #mature_w1_by_cli w1
    INNER JOIN #mature_w2_by_cli w2 ON w2.cli_id = w1.cli_id;
END
ELSE
BEGIN
    RAISERROR('Unknown @cohort_mode. Use W1_ONLY | W2_ONLY | BOTH', 16, 1);
    RETURN;
END

/* === сверочные суммы выходов по выбранному пулу (раздельно W1 и W2) === */
IF OBJECT_ID('tempdb..#maturing_check') IS NOT NULL DROP TABLE #maturing_check;
CREATE TABLE #maturing_check(
    clients_cnt int,
    deposits_maturing_sum_w1 decimal(20,4),
    deposits_maturing_sum_w2 decimal(20,4)
);
INSERT INTO #maturing_check(clients_cnt, deposits_maturing_sum_w1, deposits_maturing_sum_w2)
SELECT
    COUNT(*) AS clients_cnt,
    ISNULL( (SELECT SUM(w1.dep_sum_w1) FROM #mature_w1_by_cli w1 JOIN #clients_cohort c ON c.cli_id = w1.cli_id), 0) AS deposits_maturing_sum_w1,
    ISNULL( (SELECT SUM(w2.dep_sum_w2) FROM #mature_w2_by_cli w2 JOIN #clients_cohort c ON c.cli_id = w2.cli_id), 0) AS deposits_maturing_sum_w2;

/* === Вывод 1: пул и сверка выходов по декадам === */
SELECT
    @cohort_mode AS cohort_mode,
    clients_cnt,
    deposits_maturing_sum_w1,
    deposits_maturing_sum_w2
FROM #maturing_check;

/* ============================================================
   C) НС на 30.09 и 25.10 — только по выбранному пулу
   ============================================================*/
IF OBJECT_ID('tempdb..#ns_0930') IS NOT NULL DROP TABLE #ns_0930;
CREATE TABLE #ns_0930(cli_id bigint PRIMARY KEY, out_rub decimal(20,4));
INSERT INTO #ns_0930(cli_id, out_rub)
SELECT t.cli_id, SUM(t.out_rub)
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
JOIN #clients_cohort c ON c.cli_id = t.cli_id
WHERE
    t.dt_rep        = @dt_rep_prev
    AND t.section_name = @section_ns
    AND t.block_name   = @block_fl
    AND (@od_only = 0 OR t.od_flag = 1)
    AND t.cur          = @cur
    AND t.out_rub IS NOT NULL AND t.out_rub >= 0
GROUP BY t.cli_id;

IF OBJECT_ID('tempdb..#ns_1025') IS NOT NULL DROP TABLE #ns_1025;
CREATE TABLE #ns_1025(cli_id bigint PRIMARY KEY, out_rub decimal(20,4));
INSERT INTO #ns_1025(cli_id, out_rub)
SELECT t.cli_id, SUM(t.out_rub)
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
JOIN #clients_cohort c ON c.cli_id = t.cli_id
WHERE
    t.dt_rep        = @dt_rep_now
    AND t.section_name = @section_ns
    AND t.block_name   = @block_fl
    AND (@od_only = 0 OR t.od_flag = 1)
    AND t.cur          = @cur
    AND t.out_rub IS NOT NULL AND t.out_rub >= 0
GROUP BY t.cli_id;

/* === Вывод 2: НС было/стало (по пулу) === */
-- Было (30.09)
SELECT
    COUNT(*)     AS clients_ns_0930_cnt,
    SUM(out_rub) AS ns_0930_sum
FROM #ns_0930;

-- Стало (25.10)
SELECT
    COUNT(*)     AS clients_ns_1025_cnt,
    SUM(out_rub) AS ns_1025_sum
FROM #ns_1025;

/* ============================================================
   D) ОТКРЫТИЯ ЭТИМИ ЖЕ КЛИЕНТАМИ (по снимку 25.10), НЕ-МАРКЕТ
      + деление по РК/неРК и декадам 01–15 / 16–25
   ============================================================*/
IF OBJECT_ID('tempdb..#td_1025_base') IS NOT NULL DROP TABLE #td_1025_base;
CREATE TABLE #td_1025_base(
    dt_open_d date NOT NULL,
    con_id    bigint NOT NULL,
    cli_id    bigint NOT NULL,
    out_rub   decimal(20,4) NOT NULL,
    rate_con  decimal(9,6) NULL,
    conv      nvarchar(50) NULL
);
INSERT INTO #td_1025_base(dt_open_d, con_id, cli_id, out_rub, rate_con, conv)
SELECT
    CAST(t.dt_open AS date),
    t.con_id, t.cli_id, t.out_rub, t.rate_con, t.conv
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
JOIN #clients_cohort c ON c.cli_id = t.cli_id
LEFT JOIN #market_names m ON m.prod_name_res = t.prod_name_res
WHERE
    t.dt_rep        = @dt_rep_now
    AND t.section_name = @section_td
    AND t.block_name   = @block_fl
    AND (@od_only = 0 OR t.od_flag = 1)
    AND t.cur          = @cur
    AND t.acc_role     = @acc_role
    AND t.out_rub IS NOT NULL AND t.out_rub >= 0
    AND m.prod_name_res IS NULL
    AND CAST(t.dt_open AS date) BETWEEN @w1_from AND @w2_to;

IF OBJECT_ID('tempdb..#td_1025_by_con') IS NOT NULL DROP TABLE #td_1025_by_con;
CREATE TABLE #td_1025_by_con(
    dt_open_d date NOT NULL,
    con_id    bigint NOT NULL PRIMARY KEY,
    cli_id    bigint NOT NULL,
    out_rub   decimal(20,4) NOT NULL,
    rate_con  decimal(9,6) NULL,
    conv_norm varchar(20) NOT NULL
);
INSERT INTO #td_1025_by_con(dt_open_d, con_id, cli_id, out_rub, rate_con, conv_norm)
SELECT
    b.dt_open_d,
    b.con_id,
    MIN(b.cli_id),
    SUM(b.out_rub),
    MIN(b.rate_con),
    CASE
        WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(b.conv,''))),'')) IS NULL
            THEN 'AT_THE_END'
        ELSE UPPER(LTRIM(RTRIM(MIN(b.conv))))
    END
FROM #td_1025_base b
GROUP BY b.dt_open_d, b.con_id;

IF OBJECT_ID('tempdb..#td_1025_flag') IS NOT NULL DROP TABLE #td_1025_flag;
CREATE TABLE #td_1025_flag(
    dt_open_d date NOT NULL,
    cli_id    bigint NOT NULL,
    out_rub   decimal(20,4) NOT NULL,
    is_rk     bit NOT NULL
);
INSERT INTO #td_1025_flag(dt_open_d, cli_id, out_rub, is_rk)
SELECT
    c.dt_open_d,
    c.cli_id,
    c.out_rub,
    CASE
      WHEN c.conv_norm = 'AT_THE_END' AND EXISTS (
           SELECT 1 FROM #rk_rates rr
           WHERE rr.conv_type = 'AT_THE_END'
             AND c.dt_open_d BETWEEN rr.d_from AND rr.d_to
             AND ABS(c.rate_con - rr.r) <= @eps
      ) THEN 1
      WHEN c.conv_norm <> 'AT_THE_END' AND EXISTS (
           SELECT 1 FROM #rk_rates rr
           WHERE rr.conv_type = 'NOT_AT_THE_END'
             AND c.dt_open_d BETWEEN rr.d_from AND rr.d_to
             AND ABS(c.rate_con - rr.r) <= @eps
      ) THEN 1
      ELSE 0
    END
FROM #td_1025_by_con c;

IF OBJECT_ID('tempdb..#openings_by_client') IS NOT NULL DROP TABLE #openings_by_client;
CREATE TABLE #openings_by_client(
    window_label nvarchar(16),
    is_rk        bit,
    cli_id       bigint,
    sum_out_rub  decimal(20,4)
);
INSERT INTO #openings_by_client(window_label, is_rk, cli_id, sum_out_rub)
SELECT N'01-15 Oct', f.is_rk, f.cli_id, SUM(f.out_rub)
FROM #td_1025_flag f
WHERE f.dt_open_d BETWEEN @w1_from AND @w1_to
GROUP BY f.cli_id, f.is_rk
UNION ALL
SELECT N'16-25 Oct', f.is_rk, f.cli_id, SUM(f.out_rub)
FROM #td_1025_flag f
WHERE f.dt_open_d BETWEEN @w2_from AND @w2_to
GROUP BY f.cli_id, f.is_rk;

/* === Вывод 3–6: открытия по категориям (сумма + кол-во клиентов) === */

-- 3) Открытия 01–15 по РК
SELECT
    N'01-15 Oct' AS window_label,
    N'По РК'     AS rk_bucket,
    COUNT(*)     AS clients_cnt,
    SUM(sum_out_rub) AS deposits_opened_sum
FROM #openings_by_client
WHERE window_label = N'01-15 Oct' AND is_rk = 1;

-- 4) Открытия 01–15 НЕ по РК
SELECT
    N'01-15 Oct' AS window_label,
    N'Не по РК'  AS rk_bucket,
    COUNT(*)     AS clients_cnt,
    SUM(sum_out_rub) AS deposits_opened_sum
FROM #openings_by_client
WHERE window_label = N'01-15 Oct' AND is_rk = 0;

-- 5) Открытия 16–25 по РК
SELECT
    N'16-25 Oct' AS window_label,
    N'По РК'     AS rk_bucket,
    COUNT(*)     AS clients_cnt,
    SUM(sum_out_rub) AS deposits_opened_sum
FROM #openings_by_client
WHERE window_label = N'16-25 Oct' AND is_rk = 1;

-- 6) Открытия 16–25 НЕ по РК
SELECT
    N'16-25 Oct' AS window_label,
    N'Не по РК'  AS rk_bucket,
    COUNT(*)     AS clients_cnt,
    SUM(sum_out_rub) AS deposits_opened_sum
FROM #openings_by_client
WHERE window_label = N'16-25 Oct' AND is_rk = 0;
