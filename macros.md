USE ALM;
SET NOCOUNT ON;


/* ============================================================
   ПАРАМЕТРЫ
   ============================================================ */

DECLARE @dt_rep       date = '2026-10-03';   -- snapshot для первого отчёта
DECLARE @date_from    date = '2026-10-01';   -- открытия с
DECLARE @date_to      date = '2026-10-03';   -- открытия по

DECLARE @control_dt   date = '2026-09-30';   -- дата проверки A/B/C

DECLARE @eps decimal(9,6) = 0.0005;



/* ============================================================
   ============================================================
   1. ПЕРВЫЙ ОТЧЁТ:
      ОТКРЫТИЯ + ОБЫЧНЫЕ СРОКИ + РК

      РК определяется напрямую через:
      ALM_TEST.WORK.promo_new_money_rate_dict
   ============================================================
   ============================================================ */

;WITH base AS
(
    SELECT
          CAST(t.dt_open AS date) AS dt_open
        , TRY_CAST(t.con_id AS bigint) AS con_id
        , TRY_CAST(t.cli_id AS bigint) AS cli_id

        , t.out_rub
        , t.rate_con
        , t.rate_trf
        , t.conv
        , t.termdays

        , fk.AVG_KEY_RATE

    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)

    LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fk
        ON fk.DT_REP = CAST(t.dt_open AS date)
       AND fk.TERM   = t.termdays

    WHERE
        t.dt_rep       = @dt_rep
        AND t.section_name = N'Срочные'
        AND t.block_name   = N'Привлечение ФЛ'
        AND t.acc_role     = N'LIAB'
        AND t.od_flag      = 1
        AND t.cur          = '810'

        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0

        AND CAST(t.dt_open AS date)
            BETWEEN @date_from AND @date_to

        /* ФУ как и раньше не входят */
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
),


by_con AS
(
    SELECT
          dt_open
        , con_id
        , MIN(cli_id) AS cli_id

        , SUM(out_rub) AS out_rub
        , MIN(rate_con) AS rate_con_class
        , MIN(termdays) AS termdays

        , CASE
            WHEN MIN(
                    NULLIF(
                        LTRIM(RTRIM(COALESCE(conv,''))),
                        ''
                    )
                 ) IS NULL
                THEN 'AT_THE_END'

            ELSE UPPER(
                    LTRIM(RTRIM(MIN(conv)))
                 )
          END AS conv_norm


        /* клиентская ставка */
        , SUM(
            CASE
                WHEN rate_con IS NOT NULL
                    THEN out_rub * rate_con
            END
          ) AS wsum_rate_con

        , SUM(
            CASE
                WHEN rate_con IS NOT NULL
                    THEN out_rub
            END
          ) AS wden_rate_con


        /* ТС */
        , SUM(
            CASE
                WHEN rate_trf IS NOT NULL
                    THEN out_rub * rate_trf
            END
          ) AS wsum_rate_trf

        , SUM(
            CASE
                WHEN rate_trf IS NOT NULL
                    THEN out_rub
            END
          ) AS wden_rate_trf


        /* прогнозный КС */
        , SUM(
            CASE
                WHEN AVG_KEY_RATE IS NOT NULL
                    THEN out_rub * AVG_KEY_RATE
            END
          ) AS wsum_avg_key_rate

        , SUM(
            CASE
                WHEN AVG_KEY_RATE IS NOT NULL
                    THEN out_rub
            END
          ) AS wden_avg_key_rate

    FROM base

    GROUP BY
          dt_open
        , con_id
),


attr_ranked AS
(
    SELECT
          TRY_CAST(a.CON_ID AS bigint) AS con_id

        /* запрещённые для РК надбавки */
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
          END AS forbidden_markup

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


attr AS
(
    SELECT
          con_id
        , forbidden_markup

    FROM attr_ranked

    WHERE rn = 1
),


prepared AS
(
    SELECT
          b.*

        /* обычный бакет срока */
        , CASE
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
          END AS normal_term_bucket

        , ISNULL(a.forbidden_markup,0)
            AS forbidden_markup

    FROM by_con b

    LEFT JOIN attr a
        ON a.con_id = b.con_id
),


classified AS
(
    SELECT
          p.*

        /* если матчится справочник — берём его term_bucket */
        , d.term_bucket AS promo_term_bucket

        , CASE
            WHEN d.term_bucket IS NOT NULL
             AND p.forbidden_markup = 0
                THEN 1
            ELSE 0
          END AS is_rk

    FROM prepared p

    OUTER APPLY
    (
        SELECT TOP (1)
            r.term_bucket

        FROM ALM_TEST.WORK.promo_new_money_rate_dict r
            WITH (NOLOCK)

        WHERE
            r.is_active = 1

            /* дата открытия */
            AND p.dt_open
                BETWEEN r.date_from AND r.date_to

            /* срок */
            AND p.termdays
                BETWEEN r.term_min AND r.term_max

            /* тип выплаты */
            AND p.conv_norm = r.conv_type

            /* сумма */
            AND p.out_rub
                BETWEEN r.amount_from AND r.amount_to

            /* ставка */
            AND ABS(
                    p.rate_con_class - r.promo_rate
                ) <= @eps

        ORDER BY
              r.date_from DESC
            , r.id DESC
    ) d
),


tall AS
(
    /* ========================================================
       ОБЫЧНЫЕ ВКЛАДЫ
       ======================================================== */

    SELECT
          CAST(normal_term_bucket AS nvarchar(20))
            AS [Срок, дн.]

        , dt_open
            AS [Дата открытия]

        , SUM(out_rub)
            AS [Объем, руб.]

        , CAST(
            SUM(wsum_rate_con)
            / NULLIF(SUM(wden_rate_con),0)
            AS decimal(9,6)
          ) AS [Средневзв. ставка (клиент)]

        , CAST(
            SUM(wsum_rate_trf)
            / NULLIF(SUM(wden_rate_trf),0)
            AS decimal(9,6)
          ) AS [Средневзв. ставка (ТС)]

        , CAST(
            SUM(wsum_avg_key_rate)
            / NULLIF(SUM(wden_avg_key_rate),0)
            AS decimal(9,6)
          ) AS [Средневзв. прогнозный КС]

        , normal_term_bucket AS sort_term
        , 0 AS sort_rk

    FROM classified

    WHERE is_rk = 0

    GROUP BY
          normal_term_bucket
        , dt_open


    UNION ALL


    /* ========================================================
       РК
       ======================================================== */

    SELECT
          CONCAT(promo_term_bucket,N' РК')
            AS [Срок, дн.]

        , dt_open
            AS [Дата открытия]

        , SUM(out_rub)
            AS [Объем, руб.]

        , CAST(
            SUM(wsum_rate_con)
            / NULLIF(SUM(wden_rate_con),0)
            AS decimal(9,6)
          ) AS [Средневзв. ставка (клиент)]

        , CAST(
            SUM(wsum_rate_trf)
            / NULLIF(SUM(wden_rate_trf),0)
            AS decimal(9,6)
          ) AS [Средневзв. ставка (ТС)]

        , CAST(
            SUM(wsum_avg_key_rate)
            / NULLIF(SUM(wden_avg_key_rate),0)
            AS decimal(9,6)
          ) AS [Средневзв. прогнозный КС]

        , promo_term_bucket AS sort_term
        , 1 AS sort_rk

    FROM classified

    WHERE is_rk = 1

    GROUP BY
          promo_term_bucket
        , dt_open
)


/* ============================================================
   RESULT SET №1
   ============================================================ */

SELECT
      [Срок, дн.]
    , [Дата открытия]
    , [Объем, руб.]
    , [Средневзв. ставка (клиент)]
    , [Средневзв. ставка (ТС)]
    , [Средневзв. прогнозный КС]

FROM tall

ORDER BY
      [Дата открытия]
    , sort_term
    , sort_rk;



/* ============================================================
   ============================================================
   2. КОНТРОЛЬ A / B / C НА @control_dt

   A = НОВ / НДП / НДМ

   B = договор подходит под
       ALM_TEST.WORK.promo_new_money_rate_dict

   C = прочие надбавки:
       Пк3 Пк6 Пк7
       Пр2 Пр3
       Пк2 От1
       Мпл Пнс
   ============================================================
   ============================================================ */


IF OBJECT_ID('tempdb..#bal_check') IS NOT NULL
    DROP TABLE #bal_check;


SELECT
      TRY_CAST(t.con_id AS bigint) AS con_id

    , MIN(TRY_CAST(t.cli_id AS bigint))
        AS cli_id

    , MIN(CAST(t.dt_open AS date))
        AS dt_open

    , MIN(t.TSEGMENTNAME)
        AS TSEGMENTNAME

    , SUM(t.out_rub)
        AS out_rub

    , MIN(t.rate_con)
        AS rate_con

    , MIN(t.termdays)
        AS termdays

    , CASE
        WHEN MIN(
                NULLIF(
                    LTRIM(RTRIM(COALESCE(t.conv,''))),
                    ''
                )
             ) IS NULL
            THEN 'AT_THE_END'

        ELSE UPPER(
                LTRIM(RTRIM(MIN(t.conv)))
             )
      END AS conv_norm

INTO #bal_check

FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)

WHERE
    t.dt_rep = @control_dt

    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.od_flag      = 1
    AND t.cur          = '810'

    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0

GROUP BY
    t.con_id;


CREATE UNIQUE CLUSTERED INDEX IX_bal_check
    ON #bal_check(con_id);



/* ============================================================
   НАДБАВКИ
   ============================================================ */

IF OBJECT_ID('tempdb..#attr_check') IS NOT NULL
    DROP TABLE #attr_check;


;WITH x AS
(
    SELECT
          TRY_CAST(a.CON_ID AS bigint) AS con_id


        /* A = НОВ / НДП / НДМ */
        , CASE
            WHEN
                   ISNULL(TRY_CAST(a.[Нов] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[НДП] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[НДМ] AS int),0) = 1

                THEN 1
            ELSE 0
          END AS flag_A


        /* C = прочие надбавки */
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
          END AS flag_C


        , ROW_NUMBER() OVER
          (
              PARTITION BY a.CON_ID

              ORDER BY
                    a.DT_UPDATE DESC
                  , a.loaddate DESC
          ) AS rn

    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    INNER JOIN #bal_check b
        ON b.con_id =
           TRY_CAST(a.CON_ID AS bigint)
)

SELECT
      con_id
    , flag_A
    , flag_C

INTO #attr_check

FROM x

WHERE rn = 1;



/* ============================================================
   ФИНАЛЬНЫЕ ФЛАГИ A / B / C
   ============================================================ */

IF OBJECT_ID('tempdb..#result_check') IS NOT NULL
    DROP TABLE #result_check;


SELECT
      b.*

    , ISNULL(a.flag_A,0)
        AS flag_A

    , ISNULL(a.flag_C,0)
        AS flag_C


    /* ========================================================
       B = ПОДХОДИТ ПОД СПРАВОЧНИК
       ======================================================== */

    , CASE
        WHEN EXISTS
        (
            SELECT 1

            FROM ALM_TEST.WORK.promo_new_money_rate_dict r
                WITH (NOLOCK)

            WHERE
                r.is_active = 1

                /* дата открытия */
                AND b.dt_open
                    BETWEEN r.date_from AND r.date_to

                /* срок */
                AND b.termdays
                    BETWEEN r.term_min AND r.term_max

                /* тип выплаты */
                AND b.conv_norm = r.conv_type

                /* размер */
                AND b.out_rub
                    BETWEEN r.amount_from AND r.amount_to

                /* ставка */
                AND ABS(
                        b.rate_con - r.promo_rate
                    ) <= @eps
        )

            THEN 1
        ELSE 0

      END AS flag_B


INTO #result_check

FROM #bal_check b

LEFT JOIN #attr_check a
    ON a.con_id = b.con_id;



/* ============================================================
   RESULT SET №2
   ОБЩАЯ ПРОВЕРКА МНОЖЕСТВ
   ============================================================ */

SELECT
      SUM(out_rub)
        AS [Весь баланс]


    /* A */
    , SUM(
        CASE
            WHEN flag_A = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [A - НОВ НДП НДМ]


    /* B */
    , SUM(
        CASE
            WHEN flag_B = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [B - По справочнику ставок]


    /* A ∩ B */
    , SUM(
        CASE
            WHEN flag_A = 1
             AND flag_B = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [A ∩ B]


    /* A ∪ B */
    , SUM(
        CASE
            WHEN flag_A = 1
              OR flag_B = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [A ∪ B]


    /* A \ B */
    , SUM(
        CASE
            WHEN flag_A = 1
             AND flag_B = 0
                THEN out_rub
            ELSE 0
        END
      ) AS [A без B]


    /* B \ A */
    , SUM(
        CASE
            WHEN flag_B = 1
             AND flag_A = 0
                THEN out_rub
            ELSE 0
        END
      ) AS [B без A]


    /* ========================================================
       C = остальные надбавки
       ======================================================== */

    , SUM(
        CASE
            WHEN flag_C = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [C - Прочие надбавки]


    /* C ∩ A */
    , SUM(
        CASE
            WHEN flag_C = 1
             AND flag_A = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [C ∩ A]


    /* C ∩ B */
    , SUM(
        CASE
            WHEN flag_C = 1
             AND flag_B = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [C ∩ B]


    /* C ∩ A ∩ B */
    , SUM(
        CASE
            WHEN flag_C = 1
             AND flag_A = 1
             AND flag_B = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [C ∩ A ∩ B]

FROM #result_check;



/* ============================================================
   RESULT SET №3
   ТО ЖЕ ПО TSEGMENTNAME
   ============================================================ */

SELECT
      ISNULL(TSEGMENTNAME,N'NULL')
        AS TSEGMENTNAME

    , SUM(out_rub)
        AS [Весь баланс]

    , SUM(
        CASE WHEN flag_A = 1
             THEN out_rub ELSE 0 END
      ) AS [A - НОВ НДП НДМ]

    , SUM(
        CASE WHEN flag_B = 1
             THEN out_rub ELSE 0 END
      ) AS [B - По справочнику ставок]

    , SUM(
        CASE WHEN flag_A = 1
                  AND flag_B = 1
             THEN out_rub ELSE 0 END
      ) AS [A ∩ B]

    , SUM(
        CASE WHEN flag_A = 1
                  OR flag_B = 1
             THEN out_rub ELSE 0 END
      ) AS [A ∪ B]

    , SUM(
        CASE WHEN flag_A = 1
                  AND flag_B = 0
             THEN out_rub ELSE 0 END
      ) AS [A без B]

    , SUM(
        CASE WHEN flag_B = 1
                  AND flag_A = 0
             THEN out_rub ELSE 0 END
      ) AS [B без A]


    /* C */
    , SUM(
        CASE WHEN flag_C = 1
             THEN out_rub ELSE 0 END
      ) AS [C - Прочие надбавки]

    , SUM(
        CASE WHEN flag_C = 1
                  AND flag_A = 1
             THEN out_rub ELSE 0 END
      ) AS [C ∩ A]

    , SUM(
        CASE WHEN flag_C = 1
                  AND flag_B = 1
             THEN out_rub ELSE 0 END
      ) AS [C ∩ B]

    , SUM(
        CASE WHEN flag_C = 1
                  AND flag_A = 1
                  AND flag_B = 1
             THEN out_rub ELSE 0 END
      ) AS [C ∩ A ∩ B]

FROM #result_check

GROUP BY
    TSEGMENTNAME

ORDER BY
    TSEGMENTNAME;



/* ============================================================
   RESULT SET №4
   ПОДОГОВОРНАЯ ПРОВЕРКА
   ============================================================ */

SELECT
      con_id
    , cli_id

    , TSEGMENTNAME

    , dt_open
    , termdays

    , out_rub
    , rate_con
    , conv_norm

    , flag_A
        AS [A_НОВ_НДП_НДМ]

    , flag_B
        AS [B_СПРАВОЧНИК]

    , flag_C
        AS [C_ПРОЧИЕ_НАДБАВКИ]

FROM #result_check

WHERE
       flag_A = 1
    OR flag_B = 1
    OR flag_C = 1

ORDER BY
      TSEGMENTNAME
    , out_rub DESC;
