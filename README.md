/* ============================================================
   ПАРАМЕТРЫ
   ============================================================*/
SET NOCOUNT ON;

DECLARE @cur            varchar(3)    = '810';
DECLARE @acc_role       nvarchar(10)  = N'LIAB';
DECLARE @od_only        bit           = 1;

DECLARE @dt_rep_prev    date          = '2025-09-30';  -- снимок для Шага 1
DECLARE @dt_rep_now     date          = '2025-10-25';  -- снимок для Шага 2

-- Окна выхода (по dt_close) и открытия (по dt_open)
DECLARE @w1_from        date          = '2025-10-01';
DECLARE @w1_to          date          = '2025-10-15';
DECLARE @w2_from        date          = '2025-10-16';
DECLARE @w2_to          date          = '2025-10-25';

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
-- волна 1: с 1-го до дня перед split
(@month_start, DATEADD(day,-1,@rk_split_from), 'AT_THE_END',     0.165),
(@month_start, DATEADD(day,-1,@rk_split_from), 'AT_THE_END',     0.163),
(@month_start, DATEADD(day,-1,@rk_split_from), 'NOT_AT_THE_END', 0.162),
(@month_start, DATEADD(day,-1,@rk_split_from), 'NOT_AT_THE_END', 0.160),
-- волна 2: c split по @dt_rep_now
(@rk_split_from, @dt_rep_now, 'AT_THE_END',     0.170),
(@rk_split_from, @dt_rep_now, 'AT_THE_END',     0.168),
(@rk_split_from, @dt_rep_now, 'NOT_AT_THE_END', 0.167),
(@rk_split_from, @dt_rep_now, 'NOT_AT_THE_END', 0.165);

/* ============================================================
   ШАГ 1. СНИМОК 30.09.2025: ВЫХОДЯЩИЕ ВКЛАДЫ (НЕ-МАРКЕТ) И НС
   ============================================================*/
-- База срочных НЕ-МАРКЕТ на 30.09
IF OBJECT_ID('tempdb..#td_0930') IS NOT NULL DROP TABLE #td_0930;
CREATE TABLE #td_0930(
    con_id       bigint      NOT NULL,
    cli_id       bigint      NOT NULL,
    out_rub      decimal(20,4) NOT NULL,
    dt_close     date        NULL
);
INSERT INTO #td_0930(con_id, cli_id, out_rub, dt_close)
SELECT
    t.con_id,
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
    AND m.prod_name_res IS NULL;  -- исключили «Маркет»

-- Окно выхода 01–15
IF OBJECT_ID('tempdb..#mature_w1') IS NOT NULL DROP TABLE #mature_w1;
CREATE TABLE #mature_w1(cli_id bigint PRIMARY KEY, sum_out_rub decimal(20,4));
INSERT INTO #mature_w1(cli_id, sum_out_rub)
SELECT
    t.cli_id,
    SUM(t.out_rub) AS sum_out_rub
FROM #td_0930 t
WHERE t.dt_close BETWEEN @w1_from AND @w1_to
GROUP BY t.cli_id;

-- Окно выхода 16–25
IF OBJECT_ID('tempdb..#mature_w2') IS NOT NULL DROP TABLE #mature_w2;
CREATE TABLE #mature_w2(cli_id bigint PRIMARY KEY, sum_out_rub decimal(20,4));
INSERT INTO #mature_w2(cli_id, sum_out_rub)
SELECT
    t.cli_id,
    SUM(t.out_rub) AS sum_out_rub
FROM #td_0930 t
WHERE t.dt_close BETWEEN @w2_from AND @w2_to
GROUP BY t.cli_id;

-- Пул клиентов из обоих окон
IF OBJECT_ID('tempdb..#clients_step1') IS NOT NULL DROP TABLE #clients_step1;
CREATE TABLE #clients_step1(cli_id bigint PRIMARY KEY);
INSERT INTO #clients_step1(cli_id)
SELECT cli_id FROM #mature_w1
UNION
SELECT cli_id FROM #mature_w2;

-- НС на 30.09 (все клиенты/все НС)
IF OBJECT_ID('tempdb..#ns_0930_all') IS NOT NULL DROP TABLE #ns_0930_all;
CREATE TABLE #ns_0930_all(cli_id bigint PRIMARY KEY, out_rub decimal(20,4));
INSERT INTO #ns_0930_all(cli_id, out_rub)
SELECT
    t.cli_id,
    SUM(t.out_rub) AS out_rub
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
WHERE
    t.dt_rep        = @dt_rep_prev
    AND t.section_name = @section_ns
    AND t.block_name   = @block_fl
    AND (@od_only = 0 OR t.od_flag = 1)
    AND t.cur          = @cur
    AND t.out_rub IS NOT NULL AND t.out_rub >= 0
GROUP BY t.cli_id;

-- НС на 30.09 для клиентов из окон (пересечение)
IF OBJECT_ID('tempdb..#ns_0930_clients') IS NOT NULL DROP TABLE #ns_0930_clients;
CREATE TABLE #ns_0930_clients(cli_id bigint PRIMARY KEY, out_rub decimal(20,4));
INSERT INTO #ns_0930_clients(cli_id, out_rub)
SELECT n.cli_id, n.out_rub
FROM #ns_0930_all n
JOIN #clients_step1 c ON c.cli_id = n.cli_id;

/* ===== Вывод Шага 1 (отдельно вклады и отдельно НС) ===== */
-- 1a) Вклады, выходящие 01–15: клиенты и сумма вкладов
SELECT
    N'01-15 Oct' AS window_label,
    COUNT(*)     AS clients_maturing_cnt,
    SUM(m.sum_out_rub) AS deposits_maturing_sum
FROM #mature_w1 m;

-- 1b) Вклады, выходящие 16–25: клиенты и сумма вкладов
SELECT
    N'16-25 Oct' AS window_label,
    COUNT(*)     AS clients_maturing_cnt,
    SUM(m.sum_out_rub) AS deposits_maturing_sum
FROM #mature_w2 m;

-- 1c) НС на 30.09 (все клиенты с НС)
SELECT
    COUNT(*)          AS clients_ns_0930_cnt,
    SUM(out_rub)      AS ns_0930_sum
FROM #ns_0930_all;

-- 1d) НС на 30.09 у клиентов из окон
SELECT
    COUNT(*)          AS clients_ns_0930_from_maturing_cnt,
    SUM(out_rub)      AS ns_0930_from_maturing_sum
FROM #ns_0930_clients;

/* ============================================================
   ШАГ 2. СНИМОК 25.10.2025: НОВЫЕ ОТКРЫТИЯ У ТЕХ ЖЕ КЛИЕНТОВ
          ДЕЛЕНИЕ НА «ПО РК» И «НЕ ПО РК», + НС НА 25.10
   ============================================================*/
-- База срочных НЕ-МАРКЕТ на 25.10 по клиентам из Шага 1 и окнам 01–25.10
IF OBJECT_ID('tempdb..#td_1025_base') IS NOT NULL DROP TABLE #td_1025_base;
CREATE TABLE #td_1025_base(
    dt_open_d   date        NOT NULL,
    con_id      bigint      NOT NULL,
    cli_id      bigint      NOT NULL,
    out_rub     decimal(20,4) NOT NULL,
    rate_con    decimal(9,6) NULL,
    conv        nvarchar(50) NULL
);
INSERT INTO #td_1025_base(dt_open_d, con_id, cli_id, out_rub, rate_con, conv)
SELECT
    CAST(t.dt_open AS date) AS dt_open_d,
    t.con_id,
    t.cli_id,
    t.out_rub,
    t.rate_con,
    t.conv
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
JOIN #clients_step1 c ON c.cli_id = t.cli_id
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

-- Схлопываем по con_id и нормализуем conv
IF OBJECT_ID('tempdb..#td_1025_by_con') IS NOT NULL DROP TABLE #td_1025_by_con;
CREATE TABLE #td_1025_by_con(
    dt_open_d   date        NOT NULL,
    con_id      bigint      NOT NULL PRIMARY KEY,
    cli_id      bigint      NOT NULL,
    out_rub     decimal(20,4) NOT NULL,
    rate_con    decimal(9,6) NULL,
    conv_norm   varchar(20)  NOT NULL
);
INSERT INTO #td_1025_by_con(dt_open_d,con_id,cli_id,out_rub,rate_con,conv_norm)
SELECT
    b.dt_open_d,
    b.con_id,
    MIN(b.cli_id)   AS cli_id,
    SUM(b.out_rub)  AS out_rub,
    MIN(b.rate_con) AS rate_con,
    CASE
        WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(b.conv,''))),'')) IS NULL
             THEN 'AT_THE_END'
        ELSE UPPER(LTRIM(RTRIM(MIN(b.conv))))
    END AS conv_norm
FROM #td_1025_base b
GROUP BY b.dt_open_d, b.con_id;

-- Флаг «по РК»
IF OBJECT_ID('tempdb..#td_1025_flag') IS NOT NULL DROP TABLE #td_1025_flag;
CREATE TABLE #td_1025_flag(
    dt_open_d   date        NOT NULL,
    cli_id      bigint      NOT NULL,
    out_rub     decimal(20,4) NOT NULL,
    is_rk       bit         NOT NULL
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
    END AS is_rk
FROM #td_1025_by_con c;

-- Агрегируем на уровне клиента (чтобы считать «клиентов» и суммы по клиенту)
IF OBJECT_ID('tempdb..#td_1025_by_client') IS NOT NULL DROP TABLE #td_1025_by_client;
CREATE TABLE #td_1025_by_client(
    window_label nvarchar(16),
    cli_id       bigint,
    is_rk        bit,
    sum_out_rub  decimal(20,4)
);
INSERT INTO #td_1025_by_client(window_label, cli_id, is_rk, sum_out_rub)
SELECT N'01-15 Oct', f.cli_id, f.is_rk, SUM(f.out_rub)
FROM #td_1025_flag f
WHERE f.dt_open_d BETWEEN @w1_from AND @w1_to
GROUP BY f.cli_id, f.is_rk
UNION ALL
SELECT N'16-25 Oct', f.cli_id, f.is_rk, SUM(f.out_rub)
FROM #td_1025_flag f
WHERE f.dt_open_d BETWEEN @w2_from AND @w2_to
GROUP BY f.cli_id, f.is_rk;

-- НС на 25.10 по тем же клиентам
IF OBJECT_ID('tempdb..#ns_1025_clients') IS NOT NULL DROP TABLE #ns_1025_clients;
CREATE TABLE #ns_1025_clients(cli_id bigint PRIMARY KEY, out_rub decimal(20,4));
INSERT INTO #ns_1025_clients(cli_id, out_rub)
SELECT
    t.cli_id,
    SUM(t.out_rub) AS out_rub
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
JOIN #clients_step1 c ON c.cli_id = t.cli_id
WHERE
    t.dt_rep        = @dt_rep_now
    AND t.section_name = @section_ns
    AND t.block_name   = @block_fl
    AND (@od_only = 0 OR t.od_flag = 1)
    AND t.cur          = @cur
    AND t.out_rub IS NOT NULL AND t.out_rub >= 0
GROUP BY t.cli_id;

/* ===== Вывод Шага 2 (отдельно вклады и отдельно НС) ===== */
-- 2a) Открытия 01–15: клиенты и суммы, «по РК»
SELECT
    N'01-15 Oct' AS window_label,
    N'По РК'     AS rk_bucket,
    COUNT(*)     AS clients_cnt,
    SUM(sum_out_rub) AS deposits_opened_sum
FROM #td_1025_by_client
WHERE window_label = N'01-15 Oct' AND is_rk = 1;

-- 2b) Открытия 01–15: клиенты и суммы, «не по РК»
SELECT
    N'01-15 Oct' AS window_label,
    N'Не по РК'  AS rk_bucket,
    COUNT(*)     AS clients_cnt,
    SUM(sum_out_rub) AS deposits_opened_sum
FROM #td_1025_by_client
WHERE window_label = N'01-15 Oct' AND is_rk = 0;

-- 2c) Открытия 16–25: клиенты и суммы, «по РК»
SELECT
    N'16-25 Oct' AS window_label,
    N'По РК'     AS rk_bucket,
    COUNT(*)     AS clients_cnt,
    SUM(sum_out_rub) AS deposits_opened_sum
FROM #td_1025_by_client
WHERE window_label = N'16-25 Oct' AND is_rk = 1;

-- 2d) Открытия 16–25: клиенты и суммы, «не по РК»
SELECT
    N'16-25 Oct' AS window_label,
    N'Не по РК'  AS rk_bucket,
    COUNT(*)     AS clients_cnt,
    SUM(sum_out_rub) AS deposits_opened_sum
FROM #td_1025_by_client
WHERE window_label = N'16-25 Oct' AND is_rk = 0;

-- 2e) НС на 25.10 у этих же клиентов (кол-во клиентов и сумма НС)
SELECT
    COUNT(*)     AS clients_ns_1025_cnt,
    SUM(out_rub) AS ns_1025_sum
FROM #ns_1025_clients;
