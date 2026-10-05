/* ===== ПАРАМЕТРЫ ===== */

DECLARE @dt_rep       date          = '2026-10-03';   -- снимок баланса
DECLARE @date_from    date          = '2026-10-01';
DECLARE @date_to      date          = '2026-10-03';

DECLARE @cur          varchar(3)    = '810';
DECLARE @section_name nvarchar(50)  = N'Срочные';
DECLARE @block_name   nvarchar(100) = N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10)  = N'LIAB';
DECLARE @od_only      bit           = 1;

DECLARE @eps decimal(9,6) = 0.0005;  -- допуск по ставке ±0.0005


/* ===== Эталонные рекламные ставки с периодами действия =====
   (01–15: старые; 16–18: новые — подставь фактические периоды при переносе)
*/

IF OBJECT_ID('tempdb..#rk_rates') IS NOT NULL
    DROP TABLE #rk_rates;

CREATE TABLE #rk_rates
(
    d_from    date,
    d_to      date,
    conv_type varchar(20),   -- 'AT_THE_END' | 'NOT_AT_THE_END'
    r         decimal(9,6)
);


INSERT INTO #rk_rates(d_from, d_to, conv_type, r)
VALUES
('2026-10-01','2026-10-15','AT_THE_END',       0.145),
('2026-10-01','2026-10-15','AT_THE_END',       0.145),
('2026-10-01','2026-10-15','NOT_AT_THE_END',   0.142),
('2026-10-01','2026-10-15','NOT_AT_THE_END',   0.142),
('2026-10-16','2026-11-05','AT_THE_END',       0.145),
('2026-10-16','2026-11-05','AT_THE_END',       0.145),
('2026-10-16','2026-11-05','NOT_AT_THE_END',   0.142),
('2026-10-16','2026-11-05','NOT_AT_THE_END',   0.142);


/* ===== Основная выборка по одному снимку @dt_rep ===== */

WITH base AS
(
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

    LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fk
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

        AND t.PROD_NAME_res NOT IN
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
        )
),


/* Схлопываем договор на дату открытия
   + готовим веса для wavg(rate_con/rate_trf/avg_key_rate)
*/

by_con AS
(
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
        SUM(
            CASE
                WHEN b.rate_con IS NOT NULL
                    THEN b.out_rub * b.rate_con
            END
        ) AS wsum_rate_con,

        SUM(
            CASE
                WHEN b.rate_con IS NOT NULL
                    THEN b.out_rub
            END
        ) AS wden_rate_con,

        SUM(
            CASE
                WHEN b.rate_trf IS NOT NULL
                    THEN b.out_rub * b.rate_trf
            END
        ) AS wsum_rate_trf,

        SUM(
            CASE
                WHEN b.rate_trf IS NOT NULL
                    THEN b.out_rub
            END
        ) AS wden_rate_trf,

        SUM(
            CASE
                WHEN b.AVG_KEY_RATE IS NOT NULL
                    THEN b.out_rub * b.AVG_KEY_RATE
            END
        ) AS wsum_avg_key_rate,

        SUM(
            CASE
                WHEN b.AVG_KEY_RATE IS NOT NULL
                    THEN b.out_rub
            END
        ) AS wden_avg_key_rate

    FROM base b

    GROUP BY
        b.dt_open_d,
        b.con_id
),


/* ===== Нужные CON_ID для объекта надбавок ===== */

relevant_con_ids AS
(
    SELECT DISTINCT
        con_id
    FROM by_con
    WHERE con_id IS NOT NULL
),


/* ===== Последнее состояние надбавок по договору ===== */

attr_ranked AS
(
    SELECT
        TRY_CAST(a.CON_ID AS bigint) AS con_id,

        CASE
            WHEN
                   ISNULL(TRY_CAST(a.[Пк3] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Пк6] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Пк7] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Пр2] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Пр3] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Пк2] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[От1] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Мпл] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Пнс] AS int), 0) = 1
                THEN 1
            ELSE 0
        END AS has_forbidden_markup,

        ROW_NUMBER() OVER
        (
            PARTITION BY TRY_CAST(a.CON_ID AS bigint)
            ORDER BY
                a.DT_UPDATE DESC,
                a.loaddate  DESC
        ) AS rn

    FROM [ALM].[ehd].[attr_DepoFLConditions] a WITH (NOLOCK)

    INNER JOIN relevant_con_ids r
        ON r.con_id = TRY_CAST(a.CON_ID AS bigint)
),


attr_flags AS
(
    SELECT
        con_id,
        has_forbidden_markup
    FROM attr_ranked
    WHERE rn = 1
),


/* Маппинг срочности */

mapped AS
(
    SELECT
        b.dt_open_d,
        b.con_id,
        b.cli_id,
        b.out_rub,
        b.rate_con_class,
        b.conv_norm,

        CASE
            WHEN (b.termdays >= 28   AND b.termdays <= 44)   THEN 31
            WHEN (b.termdays >= 45   AND b.termdays <= 79)   THEN 61
            WHEN (b.termdays >= 80   AND b.termdays <= 115)  THEN 91
            WHEN (b.termdays >= 116  AND b.termdays <= 140)  THEN 124
            WHEN (b.termdays >= 141  AND b.termdays <= 174)  THEN 151
            WHEN (b.termdays >= 175  AND b.termdays <= 200)  THEN 181
            WHEN (b.termdays >= 201  AND b.termdays <= 230)  THEN 212
            WHEN (b.termdays >= 231  AND b.termdays <= 250)  THEN 243
            WHEN (b.termdays >= 251  AND b.termdays <= 290)  THEN 274
            WHEN (b.termdays >= 340  AND b.termdays <= 405)  THEN 365
            WHEN (b.termdays >= 540  AND b.termdays <= 621)  THEN 550
            WHEN (b.termdays >= 720  AND b.termdays <= 763)  THEN 750
            WHEN (b.termdays >= 1090 AND b.termdays <= 1140) THEN 1100
            WHEN (b.termdays >= 1450 AND b.termdays <= 1475) THEN 1460
            WHEN (b.termdays >= 1795 AND b.termdays <= 1830) THEN 1825
            ELSE b.termdays
        END AS term_bucket,

        ISNULL(a.has_forbidden_markup, 0) AS has_forbidden_markup,

        b.wsum_rate_con,
        b.wden_rate_con,
        b.wsum_rate_trf,
        b.wden_rate_trf,
        b.wsum_avg_key_rate,
        b.wden_avg_key_rate

    FROM by_con b

    LEFT JOIN attr_flags a
        ON a.con_id = b.con_id
),


/* Флаг 91 РК */

flag_91rk AS
(
    SELECT
        m.*,

        CASE
            WHEN m.term_bucket <> 91
                THEN 0

            /* НОВОЕ ПРАВИЛО:
               если есть Пк3 / Пк6 / Пк7 / Пр2 / Пр3 /
               Пк2 / От1 / Мпл / Пнс — это НЕ 91 РК
            */
            WHEN m.has_forbidden_markup = 1
                THEN 0

            ELSE
                CASE
                    WHEN m.conv_norm = 'AT_THE_END'
                         AND EXISTS
                         (
                             SELECT 1
                             FROM #rk_rates rr
                             WHERE rr.conv_type = 'AT_THE_END'
                               AND m.dt_open_d BETWEEN rr.d_from AND rr.d_to
                               AND ABS(m.rate_con_class - rr.r) <= @eps
                         )
                        THEN 1

                    WHEN m.conv_norm <> 'AT_THE_END'
                         AND EXISTS
                         (
                             SELECT 1
                             FROM #rk_rates rr
                             WHERE rr.conv_type = 'NOT_AT_THE_END'
                               AND m.dt_open_d BETWEEN rr.d_from AND rr.d_to
                               AND ABS(m.rate_con_class - rr.r) <= @eps
                         )
                        THEN 1

                    ELSE 0
                END

        END AS is_91_rk

    FROM mapped m
),


/* ====== ИТОГ "TALL" ======
   1) Общие сроки, НО для 91 берём только НЕ-РК
   2) Отдельной строкой — «91 РК»
*/

tall AS
(
    /* 1) Все сроки, при этом 91 — только НЕ-РК */

    SELECT
        CAST(term_bucket AS nvarchar(20)) AS [Срок, дн.],
        dt_open_d                          AS [Дата открытия],
        SUM(out_rub)                       AS [Объем, руб.],

        CAST(
            SUM(wsum_rate_con)
            / NULLIF(SUM(wden_rate_con), 0)
            AS decimal(9,6)
        ) AS [Средневзв. ставка (клиент)],

        CAST(
            SUM(wsum_rate_trf)
            / NULLIF(SUM(wden_rate_trf), 0)
            AS decimal(9,6)
        ) AS [Средневзв. ставка (ТС)],

        CAST(
            SUM(wsum_avg_key_rate)
            / NULLIF(SUM(wden_avg_key_rate), 0)
            AS decimal(9,6)
        ) AS [Средневзв. прогнозный КС]

    FROM flag_91rk

    WHERE
        term_bucket <> 91
        OR (term_bucket = 91 AND is_91_rk = 0)

    GROUP BY
        term_bucket,
        dt_open_d


    UNION ALL


    /* 2) Спец-строка: только 91 РК */

    SELECT
        N'91 РК' AS [Срок, дн.],
        f.dt_open_d AS [Дата открытия],
        SUM(f.out_rub) AS [Объем, руб.],

        CAST(
            SUM(f.wsum_rate_con)
            / NULLIF(SUM(f.wden_rate_con), 0)
            AS decimal(9,6)
        ) AS [Средневзв. ставка (клиент)],

        CAST(
            SUM(f.wsum_rate_trf)
            / NULLIF(SUM(f.wden_rate_trf), 0)
            AS decimal(9,6)
        ) AS [Средневзв. ставка (ТС)],

        CAST(
            SUM(f.wsum_avg_key_rate)
            / NULLIF(SUM(f.wden_avg_key_rate), 0)
            AS decimal(9,6)
        ) AS [Средневзв. прогнозный КС]

    FROM flag_91rk f

    WHERE
        f.term_bucket = 91
        AND f.is_91_rk = 1

    GROUP BY
        f.dt_open_d
)

SELECT *
FROM tall

ORDER BY
    [Дата открытия],
    CASE
        WHEN [Срок, дн.] = N'91 РК'
            THEN 2147483647
        ELSE TRY_CONVERT(int, [Срок, дн.])
    END,
    [Срок, дн.];
