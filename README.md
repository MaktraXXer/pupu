/* ===== ПАРАМЕТРЫ ===== */
DECLARE @dt_rep       date         = '2025-10-18';   -- один снимок баланса
DECLARE @date_from    date         = '2025-10-01';
DECLARE @date_to      date         = '2025-10-18';

DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

DECLARE @eps decimal(9,6) = 0.0005;                   -- допуск по ставке ±0.0005

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
('2025-10-01','2025-10-15','AT_THE_END',     0.165),
('2025-10-01','2025-10-15','AT_THE_END',     0.163),
('2025-10-01','2025-10-15','NOT_AT_THE_END', 0.162),
('2025-10-01','2025-10-15','NOT_AT_THE_END', 0.160),
('2025-10-16','2025-10-18','AT_THE_END',     0.170),
('2025-10-16','2025-10-18','AT_THE_END',     0.168),
('2025-10-16','2025-10-18','NOT_AT_THE_END', 0.167),
('2025-10-16','2025-10-18','NOT_AT_THE_END', 0.165);

/* ===== Основная выборка по одному снимку @dt_rep ===== */
WITH base AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.rate_trf,              -- <— добавили ТС
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
),
/* Дедуп договоров на дату открытия + подготовка весов для wavg(rate_con/rate_trf) */
by_con AS (
    SELECT
        b.dt_open_d,
        b.con_id,
        MIN(b.cli_id)    AS cli_id,
        SUM(b.out_rub)   AS out_rub,

        /* Для классификации 91 РК берём «репрезентативную» ставку на уровне договора */
        MIN(b.rate_con)  AS rate_con_class,

        /* Нормализация conv: NULL/пусто → AT_THE_END */
        CASE
            WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(b.conv,''))), '')) IS NULL
                 THEN 'AT_THE_END'
            ELSE UPPER(LTRIM(RTRIM(MIN(b.conv))))
        END              AS conv_norm,

        MIN(b.termdays)  AS termdays,

        /* Весовые суммы для корректной wavg по rate_con и rate_trf (только где ставка не NULL) */
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
        rate_con_class,  -- для классификации РК
        conv_norm,
        CASE
            WHEN (termdays>=28  AND termdays<=33)   THEN 31
            WHEN (termdays>=60  AND termdays<=70)   THEN 61
            WHEN (termdays>=80  AND termdays<=100)  THEN 91
            WHEN (termdays>=119 AND termdays<=140)  THEN 124
            WHEN (termdays>=175 AND termdays<=200)  THEN 181
            WHEN (termdays>=245 AND termdays<=290)  THEN 274
            WHEN (termdays>=340 AND termdays<=405)  THEN 365
            WHEN (termdays>=540 AND termdays<=621)  THEN 550
            WHEN (termdays>=720 AND termdays<=763)  THEN 750
            WHEN (termdays>=1090 AND termdays<=1140) THEN 1100
            WHEN (termdays>=1450 AND termdays<=1475) THEN 1460
            WHEN (termdays>=1795 AND termdays<=1830) THEN 1825
            ELSE termdays
        END AS term_bucket,

        /* протаскиваем весовые суммы/деноминаторы для дальнейшей агрегации */
        wsum_rate_con,
        wden_rate_con,
        wsum_rate_trf,
        wden_rate_trf
    FROM by_con
),
/* Флаг 91 РК: по периоду и допуску, используя rate_con_class */
flag_91rk AS (
    SELECT
        m.*,
        CASE
            WHEN m.term_bucket <> 91 THEN 0
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
        END AS is_91_rk
    FROM mapped m
),
/* Итог "tall": обычные сроки + отдельной строкой 91 РК
   Средневзвешенные считаем из сумм, где ставки НЕ NULL */
tall AS (
    /* Все сроки (в т.ч. 91 — общий) */
    SELECT
        CAST(term_bucket AS nvarchar(20)) AS [Срок, дн.],
        dt_open_d                         AS [Дата открытия],
        SUM(out_rub)                      AS [Объем, руб.],

        /* wavg client: только по строкам с rate_con NOT NULL */
        CAST(SUM(wsum_rate_con) / NULLIF(SUM(wden_rate_con),0) AS decimal(9,6)) AS [Средневзв. ставка (клиент)],

        /* wavg TRF: только по строкам с rate_trf NOT NULL */
        CAST(SUM(wsum_rate_trf) / NULLIF(SUM(wden_rate_trf),0) AS decimal(9,6)) AS [Средневзв. ставка (ТС)]
    FROM flag_91rk
    GROUP BY term_bucket, dt_open_d

    UNION ALL

    /* Спец-строка: только 91 РК */
    SELECT
        N'91 РК'                          AS [Срок, дн.],
        f.dt_open_d                       AS [Дата открытия],
        SUM(f.out_rub)                    AS [Объем, руб.],
        CAST(SUM(f.wsum_rate_con) / NULLIF(SUM(f.wden_rate_con),0) AS decimal(9,6)) AS [Средневзв. ставка (клиент)],
        CAST(SUM(f.wsum_rate_trf) / NULLIF(SUM(f.wden_rate_trf),0) AS decimal(9,6)) AS [Средневзв. ставка (ТС)]
    FROM flag_91rk f
    WHERE f.is_91_rk = 1
    GROUP BY f.dt_open_d
)
SELECT *
FROM tall
ORDER BY [Дата открытия],
         CASE WHEN [Срок, дн.] = N'91 РК' THEN 2147483647 ELSE TRY_CONVERT(int, [Срок, дн.]) END, [Срок, дн.];
