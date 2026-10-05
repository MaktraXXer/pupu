USE ALM;
SET NOCOUNT ON;

DECLARE @dt_rep date = '2026-09-30';
DECLARE @eps decimal(9,6) = 0.0005;


/* =========================================================
   СТАВКИ РК — для проверки считаем, что действуют с 01.09
   ========================================================= */

IF OBJECT_ID('tempdb..#rk_rates') IS NOT NULL DROP TABLE #rk_rates;

CREATE TABLE #rk_rates
(
    d_from    date,
    d_to      date,
    conv_type varchar(20),
    r         decimal(9,6)
);

INSERT INTO #rk_rates
VALUES
('2026-09-01','2026-09-30','AT_THE_END',     0.145),
('2026-09-01','2026-09-30','NOT_AT_THE_END', 0.142);



/* =========================================================
   БАЛАНС НА 30.09
   Берём открытия с 01.09, чтобы сравнивались одни когорты
   ========================================================= */

IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;

SELECT
      TRY_CAST(t.con_id AS bigint) AS con_id
    , MIN(TRY_CAST(t.cli_id AS bigint)) AS cli_id
    , MIN(CAST(t.dt_open AS date)) AS dt_open
    , MIN(t.TSEGMENTNAME) AS TSEGMENTNAME
    , SUM(t.out_rub) AS out_rub
    , MIN(t.rate_con) AS rate_con

    , CASE
        WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(t.conv,''))),'')) IS NULL
            THEN 'AT_THE_END'
        ELSE UPPER(LTRIM(RTRIM(MIN(t.conv))))
      END AS conv_norm

    , MIN(t.termdays) AS termdays

INTO #bal

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

    AND t.dt_open >= '2026-09-01'
    AND t.dt_open <= @dt_rep

    /* те же исключения ФУ, что были в исходном скрипте */
    AND t.PROD_NAME_res NOT IN
    (
          N'Надёжный прайм'
        , N'Надёжный VIP'
        , N'Надёжный премиум'
        , N'Надёжный промо'
        , N'Надёжный старт'
        , N'Надёжный Т2'
        , N'Надёжный Мегафон'
        , N'Надёжный процент'
        , N'Могучий'
        , N'Надёжный'
        , N'ДОМа надёжно'
        , N'Всё в ДОМ'
    )

GROUP BY
    t.con_id;

CREATE UNIQUE CLUSTERED INDEX IX_bal_con
    ON #bal(con_id);



/* =========================================================
   НАДБАВКИ — последняя запись по договору
   ========================================================= */

IF OBJECT_ID('tempdb..#attr') IS NOT NULL DROP TABLE #attr;

;WITH x AS
(
    SELECT
          TRY_CAST(a.CON_ID AS bigint) AS con_id

        /* НДП + НДМ + НОВ */
        , CASE
            WHEN
                   ISNULL(TRY_CAST(a.[НДП] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[НДМ] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Нов] AS int),0) = 1
                THEN 1
            ELSE 0
          END AS is_ndp_ndm_nov

        /* группы, которые теперь исключаем из РК */
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
          END AS is_excluded

        , ROW_NUMBER() OVER
          (
              PARTITION BY a.CON_ID
              ORDER BY
                    a.DT_UPDATE DESC
                  , a.loaddate DESC
          ) AS rn

    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    INNER JOIN #bal b
        ON b.con_id = TRY_CAST(a.CON_ID AS bigint)
)

SELECT
      con_id
    , is_ndp_ndm_nov
    , is_excluded
INTO #attr
FROM x
WHERE rn = 1;

CREATE UNIQUE CLUSTERED INDEX IX_attr_con
    ON #attr(con_id);



/* =========================================================
   РАСЧЁТ ФЛАГОВ
   ========================================================= */

;WITH flags AS
(
    SELECT
          b.*

        , ISNULL(a.is_ndp_ndm_nov,0) AS is_ndp_ndm_nov
        , ISNULL(a.is_excluded,0)    AS is_excluded

        /* подходит под условие РК по сроку + ставке */
        , CASE
            WHEN b.termdays BETWEEN 80 AND 115

             AND EXISTS
             (
                 SELECT 1
                 FROM #rk_rates r

                 WHERE b.dt_open BETWEEN r.d_from AND r.d_to

                   AND r.conv_type =
                       CASE
                           WHEN b.conv_norm = 'AT_THE_END'
                               THEN 'AT_THE_END'
                           ELSE 'NOT_AT_THE_END'
                       END

                   AND ABS(b.rate_con - r.r) <= @eps
             )

                THEN 1
            ELSE 0
          END AS rate_ok

    FROM #bal b

    LEFT JOIN #attr a
        ON a.con_id = b.con_id
)


/* =========================================================
   ИТОГ ПО TSEGMENTNAME
   ========================================================= */

SELECT
      ISNULL(TSEGMENTNAME,N'NULL') AS TSEGMENTNAME


    /* 1. НДП + НДМ + НОВ */
    , SUM(
        CASE
            WHEN is_ndp_ndm_nov = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [1_НДП_НДМ_НОВ]


    /* 2. Все подходящие по сроку + ставке */
    , SUM(
        CASE
            WHEN rate_ok = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [2_По_ставке]


    /* 3. По ставке И НЕ в исключённых группах */
    , SUM(
        CASE
            WHEN rate_ok = 1
             AND is_excluded = 0
                THEN out_rub
            ELSE 0
        END
      ) AS [3_По_ставке_без_исключенных]


    /* 4. По ставке И входят в исключённые группы */
    , SUM(
        CASE
            WHEN rate_ok = 1
             AND is_excluded = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [4_По_ставке_исключенные]


    /* твоя контрольная формула: 1 + 2 - 4 */
    , SUM(
        CASE
            WHEN is_ndp_ndm_nov = 1
                THEN out_rub
            ELSE 0
        END
      )
      +
      SUM(
        CASE
            WHEN rate_ok = 1
                THEN out_rub
            ELSE 0
        END
      )
      -
      SUM(
        CASE
            WHEN rate_ok = 1
             AND is_excluded = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [1+2-4]


FROM flags

GROUP BY
    TSEGMENTNAME

ORDER BY
    TSEGMENTNAME;
