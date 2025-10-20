/* ===== ПАРАМЕТРЫ ===== */
DECLARE @dt_rep       date         = '2025-10-18';   -- снимок
DECLARE @date_from    date         = '2025-10-16';   -- ТОЛЬКО с 16 октября
DECLARE @date_to      date         = '2025-10-18';

DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

DECLARE @eps decimal(9,6) = 0.0005; -- допуск по ставке

/* ===== РК-ставки и периоды ===== */
IF OBJECT_ID('tempdb..#rk_rates') IS NOT NULL DROP TABLE #rk_rates;
CREATE TABLE #rk_rates(
    d_from date, d_to date, conv_type varchar(20), r decimal(9,6)
);
INSERT INTO #rk_rates(d_from,d_to,conv_type,r)
VALUES
('2025-10-01','2025-10-15','AT_THE_END',     0.165),
('2025-10-01','2025-10-15','AT_THE_END',     0.163),
('2025-10-01','2025-10-15','NOT_AT_THE_END', 0.162),
('2025-10-01','2025-10-15','NOT_AT_THE_END', 0.160),
('2025-10-16','2025-10-18','AT_THE_END',     0.170),
('2025-10-16','2025-10-18','AT_THE_END',     0.168),
('2025-10-16','2025-10-18','NOT_AT_THE_END', 0.167),
('2025-10-16','2025-10-18','NOT_AT_THE_END', 0.165);

/* ===== БАЗА: один снимок, окно открытий ===== */
WITH base AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.*
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE
        t.dt_rep       = @dt_rep
        AND t.section_name = @section_name
        AND t.block_name   = @block_name
        AND (@od_only = 0 OR t.od_flag = 1)
        AND t.cur          = @cur
        AND t.acc_role     = @acc_role
        AND t.out_rub IS NOT NULL AND t.out_rub >= 0
        AND t.dt_open BETWEEN @date_from AND @date_to
),
/* Схлопываем на уровень договора в день открытия для классификации */
by_con AS (
    SELECT
        b.dt_open_d,
        b.con_id,
        MIN(b.cli_id)    AS cli_id,
        SUM(b.out_rub)   AS out_rub,
        MIN(b.rate_con)  AS rate_con,
        CASE
            WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(b.conv,''))), '')) IS NULL
                 THEN 'AT_THE_END'                           -- conv по умолчанию => ATE
            ELSE UPPER(LTRIM(RTRIM(MIN(b.conv))))
        END              AS conv_norm,
        MIN(b.termdays)  AS termdays
    FROM base b
    GROUP BY b.dt_open_d, b.con_id
),
/* Маппинг срока */
mapped AS (
    SELECT
        dt_open_d, con_id, cli_id, out_rub, rate_con, conv_norm,
        CASE
            WHEN (termdays BETWEEN 80  AND 100)  THEN 91
            WHEN (termdays BETWEEN 28  AND 33)   THEN 31
            WHEN (termdays BETWEEN 60  AND 70)   THEN 61
            WHEN (termdays BETWEEN 119 AND 140)  THEN 124
            WHEN (termdays BETWEEN 175 AND 200)  THEN 181
            WHEN (termdays BETWEEN 245 AND 290)  THEN 274
            WHEN (termdays BETWEEN 340 AND 405)  THEN 365
            WHEN (termdays BETWEEN 540 AND 621)  THEN 550
            WHEN (termdays BETWEEN 720 AND 763)  THEN 750
            WHEN (termdays BETWEEN 1090 AND 1140) THEN 1100
            WHEN (termdays BETWEEN 1450 AND 1475) THEN 1460
            WHEN (termdays BETWEEN 1795 AND 1830) THEN 1825
            ELSE termdays
        END AS term_bucket
    FROM by_con
),
/* Флаг 91 РК (по периоду и допуску по ставке) */
flag_91rk AS (
    SELECT
        m.*,
        CASE
            WHEN m.term_bucket <> 91 THEN 0
            ELSE CASE
                WHEN m.conv_norm = 'AT_THE_END'
                     AND EXISTS (
                         SELECT 1
                         FROM #rk_rates rr
                         WHERE rr.conv_type = 'AT_THE_END'
                           AND m.dt_open_d BETWEEN rr.d_from AND rr.d_to
                           AND ABS(m.rate_con - rr.r) <= @eps
                     ) THEN 1
                WHEN m.conv_norm <> 'AT_THE_END'
                     AND EXISTS (
                         SELECT 1
                         FROM #rk_rates rr
                         WHERE rr.conv_type = 'NOT_AT_THE_END'
                           AND m.dt_open_d BETWEEN rr.d_from AND rr.d_to
                           AND ABS(m.rate_con - rr.r) <= @eps
                     ) THEN 1
                ELSE 0
            END
        END AS is_91_rk
    FROM mapped m
),
/* Целевой набор договоров: 91 день, но НЕ 91 РК, и только с 16.10 */
target AS (
    SELECT con_id, dt_open_d, rate_con, conv_norm, term_bucket
    FROM flag_91rk
    WHERE term_bucket = 91
      AND is_91_rk = 0
      AND dt_open_d >= '2025-10-16'
)
/* ===== ФИНАЛ: ВЕСЬ RAW из баланса по этим договорам + атрибуты классификации ===== */
SELECT
    b.*,                                -- вся «сырая» инфа из VW_Balance_Rest_All на @dt_rep
    t.dt_open_d,
    t.term_bucket,
    t.conv_norm,
    t.rate_con      AS rate_con_by_con, -- ставка на уровне договора (из by_con)
    CASE WHEN t.term_bucket = 91 THEN 1 ELSE 0 END AS is_91_bucket,
    0 AS is_91_rk                        -- по условию выборки — это НЕ 91 РК
FROM base b
JOIN target t
  ON t.con_id = b.con_id
ORDER BY t.dt_open_d, b.con_id;
