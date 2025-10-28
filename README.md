/* ============================================================
   ПАРАМЕТРЫ
   ============================================================*/
SET NOCOUNT ON;

DECLARE @cur            varchar(3)    = '810';
DECLARE @acc_role       nvarchar(10)  = N'LIAB';
DECLARE @od_only        bit           = 1;

DECLARE @dt_rep_prev    date          = '2025-09-30';  -- снимок "до" (НС на 30.09)
DECLARE @dt_rep_now     date          = '2025-10-25';  -- снимок "после" (НС на 25.10 и открытия)

-- === ЕДИНСТВЕННОЕ окно ВЫХОДА (по dt_close), которое анализируем в этом прогоне ===
DECLARE @exit_from      date          = '2025-10-01';
DECLARE @exit_to        date          = '2025-10-15';

-- Окна ОТКРЫТИЙ (по dt_open) для разбиения
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
   A) ПУЛ КЛИЕНТОВ: ВЫХОД ВКЛАДОВ В (@exit_from; @exit_to), НЕ-МАРКЕТ
   (снимок 30.09)
   ============================================================*/
IF OBJECT_ID('tempdb..#exit_clients') IS NOT NULL DROP TABLE #exit_clients;
CREATE TABLE #exit_clients(
  cli_id       bigint PRIMARY KEY,
  sum_out_rub  decimal(20,4) NOT NULL
);

INSERT INTO #exit_clients(cli_id, sum_out_rub)
SELECT
    t.cli_id,
    SUM(t.out_rub) AS sum_out_rub      -- сумма выходящих вкладов (по клиенту)
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
    AND m.prod_name_res IS NULL
    AND TRY_CAST(t.dt_close AS date) BETWEEN @exit_from AND @exit_to
GROUP BY t.cli_id;

/* === Вывод 1: Сколько вкладов выходило (сумма) + кол-во клиентов === */
SELECT
    @exit_from AS exit_window_from,
    @exit_to   AS exit_window_to,
    COUNT(*)   AS clients_maturing_cnt,
    SUM(sum_out_rub) AS deposits_maturing_sum
FROM #exit_clients;

/* ============================================================
   B) НС на 30.09 и на 25.10 — ТОЛЬКО по этим клиентам
   ============================================================*/
-- НС на 30.09
IF OBJECT_ID('tempdb..#ns_0930_exit') IS NOT NULL DROP TABLE #ns_0930_exit;
CREATE TABLE #ns_0930_exit(cli_id bigint PRIMARY KEY, out_rub decimal(20,4));
INSERT INTO #ns_0930_exit(cli_id, out_rub)
SELECT
    t.cli_id,
    SUM(t.out_rub) AS out_rub
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
JOIN #exit_clients ec ON ec.cli_id = t.cli_id
WHERE
    t.dt_rep        = @dt_rep_prev
    AND t.section_name = @section_ns
    AND t.block_name   = @block_fl
    AND (@od_only = 0 OR t.od_flag = 1)
    AND t.cur          = @cur
    AND t.out_rub IS NOT NULL AND t.out_rub >= 0
GROUP BY t.cli_id;

-- НС на 25.10
IF OBJECT_ID('tempdb..#ns_1025_exit') IS NOT NULL DROP TABLE #ns_1025_exit;
CREATE TABLE #ns_1025_exit(cli_id bigint PRIMARY KEY, out_rub decimal(20,4));
INSERT INTO #ns_1025_exit(cli_id, out_rub)
SELECT
    t.cli_id,
    SUM(t.out_rub) AS out_rub
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
JOIN #exit_clients ec ON ec.cli_id = t.cli_id
WHERE
    t.dt_rep        = @dt_rep_now
    AND t.section_name = @section_ns
    AND t.block_name   = @block_fl
    AND (@od_only = 0 OR t.od_flag = 1)
    AND t.cur          = @cur
    AND t.out_rub IS NOT NULL AND t.out_rub >= 0
GROUP BY t.cli_id;

/* === Вывод 2: НС «Было» и «Стало» — только по пулу клиентов === */
-- Было НС на 30.09
SELECT
    COUNT(*)     AS clients_ns_0930_cnt,
    SUM(out_rub) AS ns_0930_sum
FROM #ns_0930_exit;

-- Стало НС на 25.10
SELECT
    COUNT(*)     AS clients_ns_1025_cnt,
    SUM(out_rub) AS ns_1025_sum
FROM #ns_1025_exit;

/* ============================================================
   C) ОТКРЫТИЯ ЭТИМИ ЖЕ КЛИЕНТАМИ на 25.10-СНИМКЕ:
      01–15 и 16–25 (по РК / не по РК), НЕ-МАРКЕТ
   ============================================================*/
-- База срочных на 25.10 по клиентам из пула
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
JOIN #exit_clients ec ON ec.cli_id = t.cli_id
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

-- Агрегируем на уровне клиента по окнам и РК-бакетам
IF OBJECT_ID('tempdb..#openings_by_client') IS NOT NULL DROP TABLE #openings_by_client;
CREATE TABLE #openings_by_client(
    window_label nvarchar(16),
    is_rk        bit,
    cli_id       bigint,
    sum_out_rub  decimal(20,4)
);

INSERT INTO #openings_by_client(window_label,is_rk,cli_id,sum_out_rub)
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
