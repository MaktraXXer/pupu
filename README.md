USE [ALM];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2026-03-09';   -- дата баланса
DECLARE @DateTo   date = '2026-03-14';   -- до этой даты выхода включительно

SELECT
    @DateFrom AS dt_rep,
    @DateTo   AS dt_close_plan_to,
    SUM(CAST(t.out_rub AS decimal(38,6))) AS vol_exit_deposits_rub,
    CAST(
        SUM(CASE WHEN t.rate_con IS NOT NULL THEN CAST(t.out_rub AS decimal(38,6)) * t.rate_con END)
        / NULLIF(SUM(CASE WHEN t.rate_con IS NOT NULL THEN CAST(t.out_rub AS decimal(38,6)) END), 0)
        AS decimal(18,6)
    ) AS wavg_con_rate_exit
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE 1 = 1
    AND t.dt_rep = @DateFrom
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.cur          = '810'
    AND t.od_flag      = 1
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
    AND CAST(t.dt_close_plan AS date) >  @DateFrom
    AND CAST(t.dt_close_plan AS date) <= @DateTo
    AND ISNULL(t.PROD_NAME_res, N'') NOT IN
    (
        N'Надёжный прайм',
        N'Надёжный VIP',
        N'Надёжный премиум',
        N'Надёжный промо',
        N'Надёжный старт',
        N'Надёжный Т2',
        N'Надёжный Мегафон',
        N'Надёжный процент',
        N'Могучий',
        N'Надёжный',
        N'ДОМа надёжно',
        N'Всё в ДОМ'
    );










/* ===== ПАРАМЕТРЫ ===== */
DECLARE @dt_rep       date         = '2026-03-14';   -- снимок баланса
DECLARE @date_from    date         = '2026-03-01';
DECLARE @date_to      date         = '2026-03-14';

DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

DECLARE @eps decimal(9,6) = 0.0005;  -- допуск по ставке ±0.0005

/* ===== Эталонные рекламные ставки с периодами действия =====
   (01–15: старые; 16–18: новые — подставь фактические периоды при переносе) */
IF OBJECT_ID('tempdb..#rk_rates') IS NOT NULL DROP TABLE #rk_rates;
CREATE TABLE #rk_rates(
    d_from date,
    d_to   date,
    conv_type varchar(20),   -- 'AT_THE_END' | 'NOT_AT_THE_END'
    r      decimal(9,6)
);

INSERT INTO #rk_rates(d_from,d_to,conv_type,r)
VALUES
('2026-03-01','2026-03-15','AT_THE_END',     0.160),
('2026-03-01','2026-03-15','AT_THE_END',     0.158),
('2026-03-01','2026-03-15','NOT_AT_THE_END', 0.158),
('2026-03-01','2026-03-15','NOT_AT_THE_END', 0.156),
('2026-03-16','2026-03-31','AT_THE_END',     0.160),
('2026-03-16','2026-03-31','AT_THE_END',     0.158),
('2026-03-16','2026-03-31','NOT_AT_THE_END', 0.158),
('2026-03-16','2026-03-31','NOT_AT_THE_END', 0.156);

/* ===== Основная выборка по одному снимку @dt_rep ===== */
WITH base AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.rate_trf,           -- ТС-ставка
        t.conv,
        t.termdays,
        t.prod_name_res,
        fk.AVG_KEY_RATE
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    LEFT JOIN WORK.ForecastKey_Cache fk
        ON fk.DT_REP = CAST(t.dt_open AS date)
       AND fk.TERM   = t.termdays
    WHERE
        t.dt_rep = @dt_rep
        AND t.section_name = @section_name
        AND t.block_name   = @block_name
        AND (@od_only = 0 OR t.od_flag = 1)
        AND t.cur          = @cur
        AND t.acc_role     = @acc_role
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
        AND t.dt_open BETWEEN @date_from AND @date_to
        AND t.PROD_NAME_res NOT IN (
            N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2',
            N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный',
            N'Могучий', N'ДОМа надёжно', N'Всё в ДОМ'
        )
),

/* Схлопываем договор на дату открытия + готовим веса для wavg(rate_con/rate_trf/avg_key_rate) */
by_con AS (
    SELECT
        b.dt_open_d,
        b.con_id,
        MIN(b.cli_id)  AS cli_id,
        SUM(b.out_rub) AS out_rub,

        /* Ставка для классификации РК (уровень договора) */
        MIN(b.rate_con) AS rate_con_class,

        /* Нормализация conv: NULL/пусто → AT_THE_END */
        CASE
            WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(b.conv,''))), '')) IS NULL
                 THEN 'AT_THE_END'
            ELSE UPPER(LTRIM(RTRIM(MIN(b.conv))))
        END AS conv_norm,

        MIN(b.termdays) AS termdays,

        /* Весовые суммы/деноминаторы только по строкам где ставка не NULL */
        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END) AS wsum_rate_con,
        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END)               AS wden_rate_con,

        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END) AS wsum_rate_trf,
        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub END)               AS wden_rate_trf,

        SUM(CASE WHEN b.AVG_KEY_RATE IS NOT NULL THEN b.out_rub * b.AVG_KEY_RATE END) AS wsum_avg_key_rate,
        SUM(CASE WHEN b.AVG_KEY_RATE IS NOT NULL THEN b.out_rub END)                    AS wden_avg_key_rate
    FROM base b
    GROUP BY
        b.dt_open_d,
        b.con_id
),

/* Маппинг срочности */
mapped AS (
    SELECT
        dt_open_d,
        con_id,
        cli_id,
        out_rub,
        rate_con_class,
        conv_norm,
        CASE
            WHEN (termdays >= 28   AND termdays <= 44)   THEN 31
            WHEN (termdays >= 45   AND termdays <= 79)   THEN 61
            WHEN (termdays >= 80   AND termdays <= 115)  THEN 91
            WHEN (termdays >= 116  AND termdays <= 140)  THEN 124
            WHEN (termdays >= 141  AND termdays <= 174)  THEN 151
            WHEN (termdays >= 175  AND termdays <= 200)  THEN 181
            WHEN (termdays >= 201  AND termdays <= 230)  THEN 212
            WHEN (termdays >= 231  AND termdays <= 250)  THEN 243
            WHEN (termdays >= 251  AND termdays <= 290)  THEN 274
            WHEN (termdays >= 340  AND termdays <= 405)  THEN 365
            WHEN (termdays >= 540  AND termdays <= 621)  THEN 550
            WHEN (termdays >= 720  AND termdays <= 763)  THEN 750
            WHEN (termdays >= 1090 AND termdays <= 1140) THEN 1100
            WHEN (termdays >= 1450 AND termdays <= 1475) THEN 1460
            WHEN (termdays >= 1795 AND termdays <= 1830) THEN 1825
            ELSE termdays
        END AS term_bucket,

        wsum_rate_con,
        wden_rate_con,
        wsum_rate_trf,
        wden_rate_trf,
        wsum_avg_key_rate,
        wden_avg_key_rate
    FROM by_con
),

/* Флаг 61 РК */
flag_61rk AS (
    SELECT
        m.*,
        CASE
            WHEN m.term_bucket <> 61 THEN 0
            ELSE
                CASE
                    WHEN m.conv_norm = 'AT_THE_END'
                         AND EXISTS (
                             SELECT 1
                             FROM #rk_rates rr
                             WHERE rr.conv_type = 'AT_THE_END'
                               AND m.dt_open_d BETWEEN rr.d_from AND rr.d_to
                               AND ABS(m.rate_con_class - rr.r) <= @eps
                         ) THEN 1
                    WHEN m.conv_norm <> 'AT_THE_END'
                         AND EXISTS (
                             SELECT 1
                             FROM #rk_rates rr
                             WHERE rr.conv_type = 'NOT_AT_THE_END'
                               AND m.dt_open_d BETWEEN rr.d_from AND rr.d_to
                               AND ABS(m.rate_con_class - rr.r) <= @eps
                         ) THEN 1
                    ELSE 0
                END
        END AS is_61_rk
    FROM mapped m
),

/* ====== ИТОГ "TALL" ======
   1) Общие сроки, НО для 61 берём только НЕ-РК
   2) Отдельной строкой — «61 РК»
*/
tall AS (
    /* 1) Все сроки, при этом 61 — только НЕ-РК */
    SELECT
        CAST(term_bucket AS nvarchar(20)) AS [Срок, дн.],
        dt_open_d                         AS [Дата открытия],
        SUM(out_rub)                      AS [Объем, руб.],
        CAST(SUM(wsum_rate_con) / NULLIF(SUM(wden_rate_con),0) AS decimal(9,6)) AS [Средневзв. ставка (клиент)],
        CAST(SUM(wsum_rate_trf) / NULLIF(SUM(wden_rate_trf),0) AS decimal(9,6)) AS [Средневзв. ставка (ТС)],
        CAST(SUM(wsum_avg_key_rate) / NULLIF(SUM(wden_avg_key_rate),0) AS decimal(9,6)) AS [Средневзв. прогнозный КС]
    FROM flag_61rk
    WHERE term_bucket <> 61
       OR (term_bucket = 61 AND is_61_rk = 0)
    GROUP BY
        term_bucket,
        dt_open_d

    UNION ALL

    /* 2) Спец-строка: только 61 РК */
    SELECT
        N'61 РК' AS [Срок, дн.],
        f.dt_open_d AS [Дата открытия],
        SUM(f.out_rub) AS [Объем, руб.],
        CAST(SUM(f.wsum_rate_con) / NULLIF(SUM(f.wden_rate_con),0) AS decimal(9,6)) AS [Средневзв. ставка (клиент)],
        CAST(SUM(f.wsum_rate_trf) / NULLIF(SUM(f.wden_rate_trf),0) AS decimal(9,6)) AS [Средневзв. ставка (ТС)],
        CAST(SUM(f.wsum_avg_key_rate) / NULLIF(SUM(f.wden_avg_key_rate),0) AS decimal(9,6)) AS [Средневзв. прогнозный КС]
    FROM flag_61rk f
    WHERE f.term_bucket = 61
      AND f.is_61_rk = 1
    GROUP BY
        f.dt_open_d
)
SELECT *
FROM tall
ORDER BY
    [Дата открытия],
    CASE WHEN [Срок, дн.] = N'61 РК' THEN 2147483647 ELSE TRY_CONVERT(int, [Срок, дн.]) END,
    [Срок, дн.];

    ----------




USE [ALM];
SET NOCOUNT ON;

/* ============================================
   ПАРАМЕТРЫ
   ============================================ */
DECLARE @MondayStart date = '2026-03-09';   -- обязательно понедельник
DECLARE @FactEndDate date = '2026-03-14';   -- фактическая дата конца неполной недели

/* ============================================
   ПРОВЕРКИ
   ============================================ */
IF (DATEDIFF(day, '19000101', @MondayStart) % 7) <> 0
BEGIN
    RAISERROR(N'@MondayStart должен быть понедельником.', 16, 1);
    RETURN;
END;

DECLARE @ReliableDate date = DATEADD(day, -2, CAST(GETDATE() AS date));

IF @FactEndDate > @ReliableDate
BEGIN
    RAISERROR(N'@FactEndDate не может быть больше GETDATE()-2.', 16, 1);
    RETURN;
END;

IF @FactEndDate <= @MondayStart
BEGIN
    RAISERROR(N'@FactEndDate должен быть больше @MondayStart.', 16, 1);
    RETURN;
END;

DECLARE @WeekFrom date = DATEADD(day, 1, @MondayStart);  -- вторник
DECLARE @WeekTo   date = @FactEndDate;                   -- фактический конец

IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;
IF OBJECT_ID('tempdb..#clients_scope') IS NOT NULL DROP TABLE #clients_scope;

/* ============================================
   ЗАГРУЗКА ДВУХ СРЕЗОВ БАЛАНСА:
   стартовый понедельник + фактический конец
   ============================================ */
SELECT
      CAST(t.dt_rep AS date)            AS dt_rep
    , CAST(t.cli_id AS bigint)          AS cli_id
    , CAST(t.con_id AS bigint)          AS con_id
    , CAST(t.dt_open AS date)           AS dt_open
    , CAST(t.dt_close_plan AS date)     AS dt_close_plan
    , CAST(t.section_name AS nvarchar(50)) AS section_name
    , CAST(t.out_rub AS decimal(38,6))  AS out_rub
    , CAST(t.rate_con AS decimal(18,6)) AS rate_con
    , CAST(ISNULL(t.is_floatrate,0) AS bit) AS is_floatrate
    , CAST(t.PROD_NAME_res AS nvarchar(255)) AS PROD_NAME_res
    , CAST(t.TSEGMENTNAME AS nvarchar(255))  AS TSEGMENTNAME
INTO #bal
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep IN (@MondayStart, @FactEndDate)
  AND t.section_name IN (N'Срочные', N'Накопительный счёт')
  AND t.block_name = N'Привлечение ФЛ'
  AND t.acc_role   = N'LIAB'
  AND t.cur        = '810'
  AND t.od_flag    = 1
  AND t.out_rub IS NOT NULL
  AND t.out_rub >= 0;

CREATE CLUSTERED INDEX CIX_#bal
    ON #bal (dt_rep, section_name, cli_id, con_id);

CREATE NONCLUSTERED INDEX IX_#bal_exit
    ON #bal (dt_rep, section_name, dt_close_plan, cli_id)
    INCLUDE (con_id, out_rub, rate_con, dt_open);

CREATE NONCLUSTERED INDEX IX_#bal_open
    ON #bal (dt_rep, section_name, dt_open, cli_id)
    INCLUDE (con_id, out_rub, rate_con, dt_close_plan);

CREATE NONCLUSTERED INDEX IX_#bal_ns
    ON #bal (section_name, dt_rep, cli_id)
    INCLUDE (out_rub);

/* ============================================
   КЛИЕНТЫ В СКОУПЕ: есть выходящий вклад
   ============================================ */
SELECT DISTINCT
    b.cli_id
INTO #clients_scope
FROM #bal b
WHERE b.dt_rep = @MondayStart
  AND b.section_name = N'Срочные'
  AND b.dt_close_plan >= @WeekFrom
  AND b.dt_close_plan <= @WeekTo;

CREATE UNIQUE CLUSTERED INDEX CIX_#clients_scope
    ON #clients_scope(cli_id);

/* ============================================
   ИТОГОВЫЙ РАСЧЁТ
   ============================================ */
;WITH deposits_to_exit AS
(
    SELECT
          b.cli_id
        , b.con_id
        , b.out_rub
        , b.rate_con
    FROM #bal b
    WHERE b.dt_rep = @MondayStart
      AND b.section_name = N'Срочные'
      AND b.dt_close_plan >= @WeekFrom
      AND b.dt_close_plan <= @WeekTo
),
opened_deposits AS
(
    SELECT
          b.cli_id
        , b.con_id
        , b.out_rub
        , b.rate_con
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.dt_rep = @FactEndDate
      AND b.section_name = N'Срочные'
      AND b.dt_open >= @WeekFrom
      AND b.dt_open <= @WeekTo
),
ns_by_date AS
(
    SELECT
          b.dt_rep
        , b.cli_id
        , SUM(b.out_rub) AS ns_out_rub
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.section_name = N'Накопительный счёт'
    GROUP BY
          b.dt_rep
        , b.cli_id
),
agg_exit AS
(
    SELECT
          COUNT(DISTINCT cli_id) AS cnt_cli_exit
        , COUNT(DISTINCT con_id) AS cnt_con_exit
        , SUM(out_rub) AS vol_exit
        , CAST(
            SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub * rate_con END)
            / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_exit
    FROM deposits_to_exit
),
agg_open AS
(
    SELECT
          COUNT(DISTINCT cli_id) AS cnt_cli_open
        , COUNT(DISTINCT con_id) AS cnt_con_open
        , SUM(out_rub) AS vol_open
        , CAST(
            SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub * rate_con END)
            / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_open
    FROM opened_deposits
),
agg_ns AS
(
    SELECT
          COUNT(DISTINCT c.cli_id) AS cnt_cli_scope
        , SUM(CASE WHEN n.dt_rep = @MondayStart THEN n.ns_out_rub ELSE 0 END) AS ns_start_vol
        , SUM(CASE WHEN n.dt_rep = @FactEndDate THEN n.ns_out_rub ELSE 0 END) AS ns_end_vol
    FROM #clients_scope c
    LEFT JOIN ns_by_date n
        ON c.cli_id = n.cli_id
)
SELECT
      @MondayStart AS week_start_monday
    , @FactEndDate AS fact_end_date
    , @WeekFrom AS week_from_tuesday
    , @WeekTo   AS week_to_fact_date

    , ISNULL(ns.cnt_cli_scope, 0) AS cnt_cli_scope

    , ISNULL(ex.cnt_con_exit, 0) AS cnt_con_exit
    , ISNULL(ex.vol_exit, 0)     AS vol_exit_deposits_rub
    , ex.wavg_rate_exit          AS wavg_con_rate_exit

    , ISNULL(op.cnt_con_open, 0) AS cnt_con_open
    , ISNULL(op.vol_open, 0)     AS vol_opened_deposits_rub
    , op.wavg_rate_open          AS wavg_con_rate_open

    , ISNULL(ns.ns_start_vol, 0) AS ns_balance_start_rub
    , ISNULL(ns.ns_end_vol, 0)   AS ns_balance_end_rub
    , ISNULL(ns.ns_end_vol, 0) - ISNULL(ns.ns_start_vol, 0) AS ns_balance_delta_rub
FROM agg_exit ex
CROSS JOIN agg_open op
CROSS JOIN agg_ns ns;

/*
DROP TABLE #clients_scope;
DROP TABLE #bal;
*/
