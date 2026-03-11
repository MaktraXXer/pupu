/* ===== ПАРАМЕТРЫ ===== */
DECLARE @dt_rep       date         = '2026-03-05';   -- снимок баланса
DECLARE @date_from    date         = '2026-03-01';
DECLARE @date_to      date         = '2026-03-05';

DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

DECLARE @eps decimal(9,6) = 0.0005;      -- допуск по ставке ±0.0005
DECLARE @big_check money = 1500000;      -- граница крупного чека

/* ===== Эталонные рекламные ставки с периодами действия ===== */
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
        t.rate_trf,
        t.conv,
        t.termdays,
        t.prod_name_res
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE
        t.dt_rep       = @dt_rep
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
            N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный', N'Могучий',
            N'ДОМа надёжно', N'Всё в ДОМ'
        )
),

/* Схлопываем договор на дату открытия + готовим веса для wavg(rate_con/rate_trf) */
by_con AS (
    SELECT
        b.dt_open_d,
        b.con_id,
        MIN(b.cli_id)    AS cli_id,
        SUM(b.out_rub)   AS out_rub,

        MIN(b.rate_con)  AS rate_con_class,

        CASE
            WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(b.conv,''))), '')) IS NULL
                 THEN 'AT_THE_END'
            ELSE UPPER(LTRIM(RTRIM(MIN(b.conv))))
        END AS conv_norm,

        MIN(b.termdays)  AS termdays,

        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END) AS wsum_rate_con,
        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END)               AS wden_rate_con,

        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END) AS wsum_rate_trf,
        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub END)               AS wden_rate_trf
    FROM base b
    GROUP BY b.dt_open_d, b.con_id
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
            WHEN (termdays>=28  AND termdays<=44)    THEN 31
            WHEN (termdays>=45  AND termdays<=79)    THEN 61
            WHEN (termdays>=80  AND termdays<=115)   THEN 91
            WHEN (termdays>=116 AND termdays<=140)   THEN 124
            WHEN (termdays>=141 AND termdays<=174)   THEN 151
            WHEN (termdays>=175 AND termdays<=200)   THEN 181
            WHEN (termdays>=201 AND termdays<=230)   THEN 212
            WHEN (termdays>=231 AND termdays<=250)   THEN 243
            WHEN (termdays>=251 AND termdays<=290)   THEN 274
            WHEN (termdays>=340 AND termdays<=405)   THEN 365
            WHEN (termdays>=540 AND termdays<=621)   THEN 550
            WHEN (termdays>=720 AND termdays<=763)   THEN 750
            WHEN (termdays>=1090 AND termdays<=1140) THEN 1100
            WHEN (termdays>=1450 AND termdays<=1475) THEN 1460
            WHEN (termdays>=1795 AND termdays<=1830) THEN 1825
            ELSE termdays
        END AS term_bucket,

        wsum_rate_con, wden_rate_con,
        wsum_rate_trf, wden_rate_trf
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

/* ====== ИТОГ "TALL" ====== */
tall AS (
    /* 1) Все сроки, при этом 61 — только НЕ-РК */
    SELECT
        CAST(term_bucket AS nvarchar(20)) AS [Срок, дн.],
        dt_open_d                         AS [Дата открытия],

        SUM(out_rub)                      AS [Объем, руб.],
        COUNT(*)                          AS [Кол-во con_id],
        CAST(SUM(out_rub) * 1.0 / NULLIF(COUNT(*),0) AS decimal(18,2)) AS [Средний чек, руб.],

        SUM(CASE WHEN out_rub >  @big_check THEN 1 ELSE 0 END) AS [Кол-во con_id > 1.5 млн],
        CAST(
            SUM(CASE WHEN out_rub > @big_check THEN out_rub ELSE 0 END) * 1.0
            / NULLIF(SUM(CASE WHEN out_rub > @big_check THEN 1 ELSE 0 END),0)
            AS decimal(18,2)
        ) AS [Средний чек > 1.5 млн, руб.],

        SUM(CASE WHEN out_rub <= @big_check THEN 1 ELSE 0 END) AS [Кол-во con_id <= 1.5 млн],
        CAST(
            SUM(CASE WHEN out_rub <= @big_check THEN out_rub ELSE 0 END) * 1.0
            / NULLIF(SUM(CASE WHEN out_rub <= @big_check THEN 1 ELSE 0 END),0)
            AS decimal(18,2)
        ) AS [Средний чек <= 1.5 млн, руб.],

        CAST(SUM(wsum_rate_con) / NULLIF(SUM(wden_rate_con),0) AS decimal(9,6)) AS [Средневзв. ставка (клиент)],
        CAST(SUM(wsum_rate_trf) / NULLIF(SUM(wden_rate_trf),0) AS decimal(9,6)) AS [Средневзв. ставка (ТС)]
    FROM flag_61rk
    WHERE term_bucket <> 61
       OR (term_bucket = 61 AND is_61_rk = 0)
    GROUP BY term_bucket, dt_open_d

    UNION ALL

    /* 2) Спец-строка: только 61 РК */
    SELECT
        N'61 РК'                          AS [Срок, дн.],
        f.dt_open_d                       AS [Дата открытия],

        SUM(f.out_rub)                    AS [Объем, руб.],
        COUNT(*)                          AS [Кол-во con_id],
        CAST(SUM(f.out_rub) * 1.0 / NULLIF(COUNT(*),0) AS decimal(18,2)) AS [Средний чек, руб.],

        SUM(CASE WHEN f.out_rub >  @big_check THEN 1 ELSE 0 END) AS [Кол-во con_id > 1.5 млн],
        CAST(
            SUM(CASE WHEN f.out_rub > @big_check THEN f.out_rub ELSE 0 END) * 1.0
            / NULLIF(SUM(CASE WHEN f.out_rub > @big_check THEN 1 ELSE 0 END),0)
            AS decimal(18,2)
        ) AS [Средний чек > 1.5 млн, руб.],

        SUM(CASE WHEN f.out_rub <= @big_check THEN 1 ELSE 0 END) AS [Кол-во con_id <= 1.5 млн],
        CAST(
            SUM(CASE WHEN f.out_rub <= @big_check THEN f.out_rub ELSE 0 END) * 1.0
            / NULLIF(SUM(CASE WHEN f.out_rub <= @big_check THEN 1 ELSE 0 END),0)
            AS decimal(18,2)
        ) AS [Средний чек <= 1.5 млн, руб.],

        CAST(SUM(f.wsum_rate_con) / NULLIF(SUM(f.wden_rate_con),0) AS decimal(9,6)) AS [Средневзв. ставка (клиент)],
        CAST(SUM(f.wsum_rate_trf) / NULLIF(SUM(f.wden_rate_trf),0) AS decimal(9,6)) AS [Средневзв. ставка (ТС)]
    FROM flag_61rk f
    WHERE f.term_bucket = 61
      AND f.is_61_rk = 1
    GROUP BY f.dt_open_d
)

SELECT *
FROM tall
ORDER BY [Дата открытия],
         CASE WHEN [Срок, дн.] = N'61 РК' THEN 2147483647 ELSE TRY_CONVERT(int, [Срок, дн.]) END,
         [Срок, дн.];
