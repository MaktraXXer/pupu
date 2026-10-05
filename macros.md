USE ALM;
SET NOCOUNT ON;


/* ============================================================
   ПОСЛЕДНЯЯ ДОСТУПНАЯ ДАТА ОКТЯБРЯ 2026
   ============================================================ */

DECLARE @dt_rep date;

SELECT
    @dt_rep = MAX(dt_rep)
FROM ALM.ALM.VW_Balance_Rest_All WITH (NOLOCK)
WHERE dt_rep >= '2026-10-01'
  AND dt_rep <  '2026-11-01';


SELECT @dt_rep AS dt_rep_used;


/* ============================================================
   ДОПУСК ПО СТАВКЕ РК
   ============================================================ */

DECLARE @eps decimal(9,6) = 0.0005;


/* ============================================================
   СТАВКИ 91 РК
   ============================================================ */

IF OBJECT_ID('tempdb..#rk_rates') IS NOT NULL
    DROP TABLE #rk_rates;

CREATE TABLE #rk_rates
(
    d_from    date,
    d_to      date,
    conv_type varchar(20),
    r         decimal(9,6)
);

INSERT INTO #rk_rates
(
    d_from,
    d_to,
    conv_type,
    r
)
VALUES
('2026-10-01','2026-10-15','AT_THE_END',     0.145),
('2026-10-01','2026-10-15','NOT_AT_THE_END', 0.142),
('2026-10-16','2026-11-05','AT_THE_END',     0.145),
('2026-10-16','2026-11-05','NOT_AT_THE_END', 0.142);



/* ============================================================
   БАЛАНС
   ============================================================ */

;WITH base AS
(
    SELECT
          CAST(t.dt_rep AS date)  AS dt_rep
        , CAST(t.dt_open AS date) AS dt_open
        , CAST(t.dt_close_plan AS date) AS dt_close_plan
        , TRY_CAST(t.con_id AS bigint) AS con_id
        , TRY_CAST(t.cli_id AS bigint) AS cli_id

        , t.out_rub
        , t.rate_con
        , t.rate_trf
        , t.conv
        , t.termdays
        , t.PROD_NAME_res
        , t.TSEGMENTNAME

    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)

    WHERE
        t.dt_rep = @dt_rep

        AND t.section_name = N'Срочные'
        AND t.block_name   = N'Привлечение ФЛ'
        AND t.acc_role     = N'LIAB'
        AND t.od_flag      = 1
        AND t.cur          = '810'

        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
),


/* ============================================================
   ОДНА СТРОКА НА ДОГОВОР
   ============================================================ */

by_con AS
(
    SELECT
          b.con_id
        , MIN(b.cli_id) AS cli_id

        , MIN(b.dt_open) AS dt_open
        , MIN(b.dt_close_plan) AS dt_close_plan

        , SUM(b.out_rub) AS out_rub

        , CAST(
            SUM(
                CASE
                    WHEN b.rate_con IS NOT NULL
                        THEN b.out_rub * b.rate_con
                END
            )
            /
            NULLIF(
                SUM(
                    CASE
                        WHEN b.rate_con IS NOT NULL
                            THEN b.out_rub
                    END
                ),
                0
            )
            AS decimal(9,6)
          ) AS rate_con

        , CAST(
            SUM(
                CASE
                    WHEN b.rate_trf IS NOT NULL
                        THEN b.out_rub * b.rate_trf
                END
            )
            /
            NULLIF(
                SUM(
                    CASE
                        WHEN b.rate_trf IS NOT NULL
                            THEN b.out_rub
                    END
                ),
                0
            )
            AS decimal(9,6)
          ) AS rate_trf

        , MIN(b.rate_con) AS rate_con_class

        , CASE
            WHEN MIN(
                    NULLIF(
                        LTRIM(RTRIM(COALESCE(b.conv,''))),
                        ''
                    )
                 ) IS NULL
                THEN 'AT_THE_END'

            ELSE UPPER(
                    LTRIM(
                        RTRIM(MIN(b.conv))
                    )
                 )
          END AS conv_norm

        , MIN(b.termdays) AS termdays

        , MIN(b.PROD_NAME_res) AS PROD_NAME_res
        , MIN(b.TSEGMENTNAME)  AS TSEGMENTNAME

    FROM base b

    GROUP BY
        b.con_id
),


/* ============================================================
   ПОСЛЕДНЯЯ ЗАПИСЬ ПО НАДБАВКАМ
   ============================================================ */

attr_ranked AS
(
    SELECT
          TRY_CAST(a.CON_ID AS bigint) AS con_id

        , CASE
            WHEN
                   ISNULL(TRY_CAST(a.[Пк3] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пк6] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пк7] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пр2] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пр3] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пк2] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[От1] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Мпл] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пнс] AS int),0) = 1
                THEN 1
            ELSE 0
          END AS has_forbidden_markup

        , ROW_NUMBER() OVER
          (
              PARTITION BY a.CON_ID
              ORDER BY
                    a.DT_UPDATE DESC
                  , a.loaddate DESC
          ) AS rn

    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    INNER JOIN by_con b
        ON b.con_id = TRY_CAST(a.CON_ID AS bigint)
),


attrs AS
(
    SELECT
          con_id
        , has_forbidden_markup

    FROM attr_ranked

    WHERE rn = 1
),


/* ============================================================
   СРОЧНОСТЬ + ФЛАГ ФУ
   ============================================================ */

mapped AS
(
    SELECT
          b.*

        , CASE
            WHEN b.termdays BETWEEN 28   AND 44   THEN 31
            WHEN b.termdays BETWEEN 45   AND 79   THEN 61
            WHEN b.termdays BETWEEN 80   AND 115  THEN 91
            WHEN b.termdays BETWEEN 116  AND 140  THEN 124
            WHEN b.termdays BETWEEN 141  AND 174  THEN 151
            WHEN b.termdays BETWEEN 175  AND 200  THEN 181
            WHEN b.termdays BETWEEN 201  AND 230  THEN 212
            WHEN b.termdays BETWEEN 231  AND 250  THEN 243
            WHEN b.termdays BETWEEN 251  AND 290  THEN 274
            WHEN b.termdays BETWEEN 340  AND 405  THEN 365
            WHEN b.termdays BETWEEN 540  AND 621  THEN 550
            WHEN b.termdays BETWEEN 720  AND 763  THEN 750
            WHEN b.termdays BETWEEN 1090 AND 1140 THEN 1100
            WHEN b.termdays BETWEEN 1450 AND 1475 THEN 1460
            WHEN b.termdays BETWEEN 1795 AND 1830 THEN 1825
            ELSE b.termdays
          END AS term_bucket

        , CASE
            WHEN b.PROD_NAME_res IN
            (
                  N'Надёжный прайм'
                , N'Надёжный VIP'
                , N'Надёжный премиум'
                , N'Надёжный промо'
                , N'Надёжный старт'
                , N'Надёжный Т2'
                , N'Надёжный Мегафон'
                , N'Надёжный процент'
                , N'Надёжныйпроцент'
                , N'Могучий'
                , N'Надёжный'
                , N'ДОМа надёжно'
                , N'Всё в ДОМ'
            )
                THEN 1
            ELSE 0
          END AS is_fu

        , ISNULL(a.has_forbidden_markup,0)
            AS has_forbidden_markup

    FROM by_con b

    LEFT JOIN attrs a
        ON a.con_id = b.con_id
),


/* ============================================================
   КАТЕГОРИЯ
   ============================================================ */

classified AS
(
    SELECT
        m.*,

        CASE

            /* 91 РК */
            WHEN m.term_bucket = 91

             /* ФУ не относим к РК */
             AND m.is_fu = 0

             /* запрещённых надбавок быть не должно */
             AND m.has_forbidden_markup = 0

             AND
             (
                    (
                        m.conv_norm = 'AT_THE_END'

                        AND EXISTS
                        (
                            SELECT 1
                            FROM #rk_rates r

                            WHERE
                                r.conv_type = 'AT_THE_END'
                                AND m.dt_open
                                    BETWEEN r.d_from AND r.d_to
                                AND ABS(
                                    m.rate_con_class - r.r
                                ) <= @eps
                        )
                    )

                    OR

                    (
                        m.conv_norm <> 'AT_THE_END'

                        AND EXISTS
                        (
                            SELECT 1
                            FROM #rk_rates r

                            WHERE
                                r.conv_type = 'NOT_AT_THE_END'
                                AND m.dt_open
                                    BETWEEN r.d_from AND r.d_to
                                AND ABS(
                                    m.rate_con_class - r.r
                                ) <= @eps
                        )
                    )
             )

                THEN N'91 РК'


            /* все остальные — обычный бакет */
            ELSE CAST(
                    m.term_bucket
                    AS nvarchar(20)
                 )

        END AS category

    FROM mapped m
)


/* ============================================================
   РЕЗУЛЬТАТ — ВСЕ ВКЛАДЫ
   ============================================================ */

SELECT
      @dt_rep AS dt_rep
    , con_id
    , cli_id

    , dt_open
    , dt_close_plan

    , out_rub

    , rate_con
    , rate_trf

    , termdays
    , conv_norm AS conv

    , PROD_NAME_res
    , TSEGMENTNAME

    , category AS [Категория]

FROM classified

ORDER BY
      CASE
          WHEN category = N'91 РК'
              THEN 91
          ELSE TRY_CONVERT(int, category)
      END
    , category
    , out_rub DESC;
