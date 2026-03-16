USE [ALM];
SET NOCOUNT ON;

/* =========================================================
   ПАРАМЕТРЫ
   начало недели = первый понедельник
   конец недели  = следующий понедельник
   логика недели = вторник .. следующий понедельник
   ========================================================= */
DECLARE @MondayStart date = '2026-02-23';
DECLARE @MondayEnd   date = '2026-03-02';

DECLARE @WeekFrom date = DATEADD(day, 1, @MondayStart); -- вторник
DECLARE @WeekTo   date = @MondayEnd;                    -- понедельник

IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;
IF OBJECT_ID('tempdb..#clients_scope') IS NOT NULL DROP TABLE #clients_scope;

/* =========================================================
   1. ОДИН РАЗ ЧИТАЕМ БАЛАНС
   Сразу:
   - только 2 даты
   - только 2 секции
   - только нужные фильтры
   ========================================================= */
SELECT
    t.dt_rep,
    t.cli_id,
    t.con_id,
    CAST(t.dt_open AS date)       AS dt_open,
    CAST(t.dt_close_plan AS date) AS dt_close_plan,
    t.section_name,
    t.out_rub,
    t.rate_con,
    ISNULL(t.is_floatrate, 0)     AS is_floatrate,
    t.PROD_NAME_res,
    t.TSEGMENTNAME
INTO #bal
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE 1 = 1
    AND t.dt_rep IN (@MondayStart, @MondayEnd)
    AND t.section_name IN (N'Срочные', N'Накопительный счёт')
    AND t.block_name = N'Привлечение ФЛ'
    AND t.acc_role   = N'LIAB'
    AND t.cur        = '810'
    AND t.od_flag    = 1
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;

/* =========================================================
   2. ИНДЕКСЫ НА #bal
   Делаем не один абстрактный индекс, а под сценарии:
   - выходящие вклады
   - новые вклады
   - остатки НС
   ========================================================= */

/* Базовый индекс под дату/секцию/клиента */
CREATE CLUSTERED INDEX CIX_#bal
    ON #bal (dt_rep, section_name, cli_id, con_id);

/* Под поиск вкладов к выходу */
CREATE NONCLUSTERED INDEX IX_#bal_exit
    ON #bal (dt_rep, section_name, dt_close_plan, cli_id)
    INCLUDE (con_id, out_rub, rate_con, dt_open);

/* Под поиск новых вкладов */
CREATE NONCLUSTERED INDEX IX_#bal_open
    ON #bal (dt_rep, section_name, dt_open, cli_id)
    INCLUDE (con_id, out_rub, rate_con, dt_close_plan);

/* Под НС по клиенту на дату */
CREATE NONCLUSTERED INDEX IX_#bal_ns
    ON #bal (section_name, dt_rep, cli_id)
    INCLUDE (out_rub);

/* =========================================================
   3. КЛИЕНТЫ ИЗ СКОУПА
   Это клиенты, у которых на первый понедельник есть вклады,
   выходящие по плановой дате во вторник..понедельник
   ========================================================= */
SELECT DISTINCT
    b.cli_id
INTO #clients_scope
FROM #bal b
WHERE 1 = 1
    AND b.dt_rep = @MondayStart
    AND b.section_name = N'Срочные'
    AND b.dt_close_plan >= @WeekFrom
    AND b.dt_close_plan <= @WeekTo;

CREATE UNIQUE CLUSTERED INDEX CIX_#clients_scope
    ON #clients_scope(cli_id);

/* =========================================================
   4. ИТОГОВЫЙ РАСЧЕТ
   ========================================================= */
;WITH deposits_to_exit AS (
    SELECT
        b.cli_id,
        b.con_id,
        b.out_rub,
        b.rate_con
    FROM #bal b
    WHERE 1 = 1
        AND b.dt_rep = @MondayStart
        AND b.section_name = N'Срочные'
        AND b.dt_close_plan >= @WeekFrom
        AND b.dt_close_plan <= @WeekTo
),
opened_deposits AS (
    SELECT
        b.cli_id,
        b.con_id,
        b.out_rub,
        b.rate_con
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE 1 = 1
        AND b.dt_rep = @MondayEnd
        AND b.section_name = N'Срочные'
        AND b.dt_open >= @WeekFrom
        AND b.dt_open <= @WeekTo
),
ns_by_date AS (
    SELECT
        b.dt_rep,
        b.cli_id,
        SUM(CAST(b.out_rub AS decimal(38,6))) AS ns_out_rub
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE 1 = 1
        AND b.section_name = N'Накопительный счёт'
    GROUP BY
        b.dt_rep,
        b.cli_id
),
agg_exit AS (
    SELECT
        COUNT(DISTINCT cli_id) AS cnt_cli_exit,
        COUNT(DISTINCT con_id) AS cnt_con_exit,
        SUM(CAST(out_rub AS decimal(38,6))) AS vol_exit,
        CAST(
            SUM(CASE WHEN rate_con IS NOT NULL THEN CAST(out_rub AS decimal(38,6)) * rate_con END)
            / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL THEN CAST(out_rub AS decimal(38,6)) END), 0)
            AS decimal(18,6)
        ) AS wavg_rate_exit
    FROM deposits_to_exit
),
agg_open AS (
    SELECT
        COUNT(DISTINCT cli_id) AS cnt_cli_open,
        COUNT(DISTINCT con_id) AS cnt_con_open,
        SUM(CAST(out_rub AS decimal(38,6))) AS vol_open,
        CAST(
            SUM(CASE WHEN rate_con IS NOT NULL THEN CAST(out_rub AS decimal(38,6)) * rate_con END)
            / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL THEN CAST(out_rub AS decimal(38,6)) END), 0)
            AS decimal(18,6)
        ) AS wavg_rate_open
    FROM opened_deposits
),
agg_ns AS (
    SELECT
        COUNT(DISTINCT c.cli_id) AS cnt_cli_scope,
        SUM(CASE WHEN n.dt_rep = @MondayStart THEN n.ns_out_rub ELSE 0 END) AS ns_start_vol,
        SUM(CASE WHEN n.dt_rep = @MondayEnd   THEN n.ns_out_rub ELSE 0 END) AS ns_end_vol
    FROM #clients_scope c
    LEFT JOIN ns_by_date n
        ON c.cli_id = n.cli_id
)
SELECT
    @MondayStart AS week_start_monday,
    @MondayEnd   AS week_end_monday,
    @WeekFrom    AS week_from_tuesday,
    @WeekTo      AS week_to_monday,

    ns.cnt_cli_scope                AS cnt_cli_scope,

    ex.cnt_con_exit                 AS cnt_con_exit,
    ex.vol_exit                     AS vol_exit_deposits_rub,
    ex.wavg_rate_exit               AS wavg_con_rate_exit,

    op.cnt_con_open                 AS cnt_con_open,
    op.vol_open                     AS vol_opened_deposits_rub,
    op.wavg_rate_open               AS wavg_con_rate_open,

    ns.ns_start_vol                 AS ns_balance_start_rub,
    ns.ns_end_vol                   AS ns_balance_end_rub,
    ns.ns_end_vol - ns.ns_start_vol AS ns_balance_delta_rub
FROM agg_exit ex
CROSS JOIN agg_open op
CROSS JOIN agg_ns ns;

/* =========================================================
   5. ОТЛАДОЧНЫЕ ПРОВЕРКИ
   Их удобно оставить на время теста
   ========================================================= */
SELECT COUNT(*) AS rows_in_#bal FROM #bal;
SELECT COUNT(*) AS cnt_clients_scope FROM #clients_scope;

/*
DROP TABLE #clients_scope;
DROP TABLE #bal;
*/
