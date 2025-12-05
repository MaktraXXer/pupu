/* ======================= ПАРАМЕТРЫ ======================= */
DECLARE @dt_rep_from  date       = '2025-11-28';   -- начало периода снимков
DECLARE @dt_rep_to    date       = '2025-12-03';   -- конец периода снимков
DECLARE @date_from    date       = '2025-11-28';   -- период открытия договоров
DECLARE @date_to      date       = '2025-12-03';

DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

DECLARE @eps decimal(9,6) = 0.0005;                 -- допуск по ставке ±0.0005

/* ===== Эталонные рекламные ставки 61 дней (РК) ===== */
IF OBJECT_ID('tempdb..#rk_rates') IS NOT NULL DROP TABLE #rk_rates;
CREATE TABLE #rk_rates(
    d_from    date,
    d_to      date,
    conv_type varchar(20),   -- 'AT_THE_END' | 'NOT_AT_THE_END'
    r         decimal(9,6)
);

INSERT INTO #rk_rates(d_from,d_to,conv_type,r)
VALUES
('2025-11-28','2025-12-03','AT_THE_END',     0.170),
('2025-11-28','2025-12-03','AT_THE_END',     0.168),
('2025-11-28','2025-12-03','NOT_AT_THE_END', 0.168),
('2025-11-28','2025-12-03','NOT_AT_THE_END', 0.166);

/* ======================= ОСНОВНАЯ ВЫБОРКА ======================= */
WITH base AS (
    SELECT
        CAST(t.dt_rep  AS date) AS dt_rep_d,    -- дата снимка баланса
        CAST(t.dt_open AS date) AS dt_open_d,   -- дата открытия договора
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.rate_trf,              -- ТС-ставка
        t.conv,
        t.termdays,
        t.prod_name_res
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE
        t.dt_rep BETWEEN @dt_rep_from AND @dt_rep_to
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
               N'ДОМа надёжно', N'Всё в ДОМ', N'Могучий'
        )
),
/* Схлопываем договор на дату снимка + дату открытия, готовим веса для wavg(rate_con/rate_trf) */
by_con AS (
    SELECT
        b.dt_rep_d,
        b.dt_open_d,
        b.con_id,
        MIN(b.cli_id)    AS cli_id,
        SUM(b.out_rub)   AS out_rub,

        /* Ставка для классификации РК (уровень договора) */
        MIN(b.rate_con)  AS rate_con_class,

        /* Нормализация conv: NULL/пусто → AT_THE_END */
        CASE
            WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(b.conv,''))), '')) IS NULL
                 THEN 'AT_THE_END'
            ELSE UPPER(LTRIM(RTRIM(MIN(b.conv))))
        END              AS conv_norm,

        MIN(b.termdays)  AS termdays,

        /* Весовые суммы/деноминаторы только по строкам, где ставка не NULL */
        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END) AS wsum_rate_con,
        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END)               AS wden_rate_con,

        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END) AS wsum_rate_trf,
        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub END)               AS wden_rate_trf
    FROM base b
    GROUP BY b.dt_rep_d, b.dt_open_d, b.con_id
),
/* Маппинг срочности по расширенному справочнику */
mapped AS (
    SELECT
        dt_rep_d,
        dt_open_d,
        con_id,
        cli_id,
        out_rub,
        rate_con_class,
        conv_norm,
        CASE
            WHEN termdays BETWEEN  28 AND  44  THEN  31
            WHEN termdays BETWEEN  45 AND  70  THEN  61
            WHEN termdays BETWEEN  85 AND 110  THEN  91
            WHEN termdays BETWEEN 119 AND 140  THEN 124
            WHEN termdays BETWEEN 141 AND 170  THEN 151
            WHEN termdays BETWEEN 175 AND 200  THEN 181
            WHEN termdays BETWEEN 201 AND 230  THEN 212
            WHEN termdays BETWEEN 231 AND 250  THEN 243
            WHEN termdays BETWEEN 251 AND 290  THEN 274
            WHEN termdays BETWEEN 340 AND 405  THEN 365
            WHEN termdays BETWEEN 540 AND 621  THEN 550
            WHEN termdays BETWEEN 720 AND 763  THEN 750
            WHEN termdays BETWEEN 1090 AND 1140 THEN 1100
            WHEN termdays BETWEEN 1450 AND 1475 THEN 1460
            WHEN termdays BETWEEN 1795 AND 1830 THEN 1825
            ELSE termdays
        END AS term_bucket,

        wsum_rate_con, wden_rate_con,
        wsum_rate_trf, wden_rate_trf
    FROM by_con
),
/* Флаг «61 РК» (рекламный продукт на 61 день) */
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
/* ===== ИТОГ "TALL"
   1) Все сроки, при этом 61 — только НЕ-РК (is_61_rk=0)
   2) Отдельной строкой — «61 РК» (is_61_rk=1)
*/
tall AS (
    /* 1) Общие сроки, 61 без РК */
    SELECT
        CAST(f.term_bucket AS nvarchar(20)) AS [Срок, дн.],
        f.dt_rep_d                          AS [Дата снимка],
        f.dt_open_d                         AS [Дата открытия],
        SUM(f.out_rub)                      AS [Объем, руб.],
        CAST(SUM(f.wsum_rate_con) / NULLIF(SUM(f.wden_rate_con),0) AS decimal(9,6)) AS [Средневзв. ставка (клиент)],
        CAST(SUM(f.wsum_rate_trf) / NULLIF(SUM(f.wden_rate_trf),0) AS decimal(9,6)) AS [Средневзв. ставка (ТС)]
    FROM flag_61rk f
    WHERE f.term_bucket <> 61        -- все сроки кроме 61
       OR (f.term_bucket = 61 AND f.is_61_rk = 0)  -- 61 только НЕ-РК
    GROUP BY f.term_bucket, f.dt_rep_d, f.dt_open_d

    UNION ALL

    /* 2) Спец-строка: только 61 РК */
    SELECT
        N'61 РК'                          AS [Срок, дн.],
        f.dt_rep_d                        AS [Дата снимка],
        f.dt_open_d                       AS [Дата открытия],
        SUM(f.out_rub)                    AS [Объем, руб.],
        CAST(SUM(f.wsum_rate_con) / NULLIF(SUM(f.wden_rate_con),0) AS decimal(9,6)) AS [Средневзв. ставка (клиент)],
        CAST(SUM(f.wsum_rate_trf) / NULLIF(SUM(f.wden_rate_trf),0) AS decimal(9,6)) AS [Средневзв. ставка (ТС)]
    FROM flag_61rk f
    WHERE f.term_bucket = 61
      AND f.is_61_rk = 1
    GROUP BY f.dt_rep_d, f.dt_open_d
)
SELECT *
FROM tall
ORDER BY
    [Дата снимка],
    [Дата открытия],
    CASE WHEN [Срок, дн.] = N'61 РК' THEN 2147483647 ELSE TRY_CONVERT(int, [Срок, дн.]) END,
    [Срок, дн.];
