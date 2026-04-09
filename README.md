/* ===== ПАРАМЕТРЫ ===== */
DECLARE @dt_rep       date         = '2026-04-07';
DECLARE @date_from    date         = '2026-03-01';
DECLARE @date_to      date         = '2026-04-07';

DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

DECLARE @eps decimal(9,6) = 0.0005;

/* ===== РК ставки ===== */
IF OBJECT_ID('tempdb..#rk_rates') IS NOT NULL DROP TABLE #rk_rates;
CREATE TABLE #rk_rates(
    d_from date,
    d_to   date,
    conv_type varchar(20),
    r      decimal(9,6)
);

INSERT INTO #rk_rates(d_from,d_to,conv_type,r)
VALUES
('2026-03-01','2026-03-15','AT_THE_END',     0.160),
('2026-03-01','2026-03-15','AT_THE_END',     0.158),
('2026-03-01','2026-03-15','NOT_AT_THE_END', 0.158),
('2026-03-01','2026-03-15','NOT_AT_THE_END', 0.156),
('2026-03-16','2026-04-07','AT_THE_END',     0.160),
('2026-03-16','2026-04-07','AT_THE_END',     0.158),
('2026-03-16','2026-04-07','NOT_AT_THE_END', 0.158),
('2026-03-16','2026-04-07','NOT_AT_THE_END', 0.156);

WITH base AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.conv,
        t.termdays
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
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
by_con AS (
    SELECT
        b.dt_open_d,
        b.con_id,
        MIN(b.cli_id) AS cli_id,
        SUM(b.out_rub) AS out_rub,
        MIN(b.rate_con) AS rate_con_class,
        CASE
            WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(b.conv,''))), '')) IS NULL THEN 'AT_THE_END'
            ELSE UPPER(LTRIM(RTRIM(MIN(b.conv))))
        END AS conv_norm,
        MIN(b.termdays) AS termdays
    FROM base b
    GROUP BY
        b.dt_open_d,
        b.con_id
),
mapped AS (
    SELECT
        dt_open_d,
        con_id,
        cli_id,
        out_rub,
        rate_con_class,
        conv_norm,
        CASE
            WHEN termdays BETWEEN 28   AND 44   THEN 31
            WHEN termdays BETWEEN 45   AND 79   THEN 61
            WHEN termdays BETWEEN 80   AND 115  THEN 91
            WHEN termdays BETWEEN 116  AND 140  THEN 124
            WHEN termdays BETWEEN 141  AND 174  THEN 151
            WHEN termdays BETWEEN 175  AND 200  THEN 181
            WHEN termdays BETWEEN 201  AND 230  THEN 212
            WHEN termdays BETWEEN 231  AND 250  THEN 243
            WHEN termdays BETWEEN 251  AND 290  THEN 274
            WHEN termdays BETWEEN 340  AND 405  THEN 365
            WHEN termdays BETWEEN 540  AND 621  THEN 550
            WHEN termdays BETWEEN 720  AND 763  THEN 750
            WHEN termdays BETWEEN 1090 AND 1140 THEN 1100
            WHEN termdays BETWEEN 1450 AND 1475 THEN 1460
            WHEN termdays BETWEEN 1795 AND 1830 THEN 1825
            ELSE termdays
        END AS term_bucket
    FROM by_con
),
flag_61rk AS (
    SELECT
        m.*,
        CASE
            WHEN m.term_bucket <> 61 THEN 0
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
        END AS is_61_rk
    FROM mapped m
)
SELECT
    SUM(out_rub) AS vol_61_rk_rub,
    COUNT(DISTINCT cli_id) AS cnt_cli_61_rk,
    CAST(
        SUM(out_rub) / NULLIF(COUNT(DISTINCT cli_id), 0)
        AS decimal(18,2)
    ) AS avg_deposit_per_client_61_rk_rub
FROM flag_61rk
WHERE term_bucket = 61
  AND is_61_rk = 1;
