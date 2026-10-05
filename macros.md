USE ALM;
SET NOCOUNT ON;

DECLARE @dt_rep     date = '2026-10-03';
DECLARE @date_from  date = '2026-10-01';
DECLARE @date_to    date = '2026-10-03';

DECLARE @control_dt date = '2026-09-30';

DECLARE @eps decimal(9,6) = 0.0005;


/* ============================================================
   ЧАСТЬ 1
   НОВЫЕ ПРИВЛЕЧЕНИЯ
   ============================================================ */

IF OBJECT_ID('tempdb..#main_bal') IS NOT NULL DROP TABLE #main_bal;

SELECT
      TRY_CAST(t.con_id AS bigint) AS con_id
    , MIN(TRY_CAST(t.cli_id AS bigint)) AS cli_id
    , CAST(t.dt_open AS date) AS dt_open

    , MIN(t.TSEGMENTNAME)  AS TSEGMENTNAME
    , MIN(t.PROD_NAME_res) AS PROD_NAME_res

    , SUM(t.out_rub) AS out_rub

    , MIN(t.rate_con) AS rate_con_class
    , MIN(t.termdays) AS termdays

    , CASE
        WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(t.conv,''))),'')) IS NULL
            THEN 'AT_THE_END'
        ELSE UPPER(MIN(NULLIF(LTRIM(RTRIM(COALESCE(t.conv,''))),'')))
      END AS conv_norm


    /* веса клиентской ставки */
    , SUM(CASE
            WHEN t.rate_con IS NOT NULL
            THEN t.out_rub * t.rate_con
          END) AS wsum_rate_con

    , SUM(CASE
            WHEN t.rate_con IS NOT NULL
            THEN t.out_rub
          END) AS wden_rate_con


    /* веса ТС */
    , SUM(CASE
            WHEN t.rate_trf IS NOT NULL
            THEN t.out_rub * t.rate_trf
          END) AS wsum_rate_trf

    , SUM(CASE
            WHEN t.rate_trf IS NOT NULL
            THEN t.out_rub
          END) AS wden_rate_trf


    /* веса прогнозного КС */
    , SUM(CASE
            WHEN fk.AVG_KEY_RATE IS NOT NULL
            THEN t.out_rub * fk.AVG_KEY_RATE
          END) AS wsum_avg_key_rate

    , SUM(CASE
            WHEN fk.AVG_KEY_RATE IS NOT NULL
            THEN t.out_rub
          END) AS wden_avg_key_rate

INTO #main_bal

FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)

LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fk
    ON fk.DT_REP = CAST(t.dt_open AS date)
   AND fk.TERM   = t.termdays

WHERE
    t.dt_rep = @dt_rep

    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.od_flag      = 1
    AND t.cur          = '810'

    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0

    AND CAST(t.dt_open AS date)
        BETWEEN @date_from AND @date_to

GROUP BY
      t.con_id
    , CAST(t.dt_open AS date);


CREATE UNIQUE CLUSTERED INDEX IX_main_bal
    ON #main_bal(con_id);



/* ============================================================
   НАДБАВКИ
   ============================================================ */

IF OBJECT_ID('tempdb..#main_attr') IS NOT NULL DROP TABLE #main_attr;

;WITH x AS
(
    SELECT
          TRY_CAST(a.CON_ID AS bigint) AS con_id

        /* новые деньги */
        , CASE
            WHEN
                   ISNULL(TRY_CAST(a.[Нов] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[НДП] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[НДМ] AS int),0) = 1
            THEN 1 ELSE 0
          END AS flag_newmoney


        /* прочие надбавки */
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
            THEN 1 ELSE 0
          END AS flag_other_markup


        , ROW_NUMBER() OVER
          (
              PARTITION BY a.CON_ID
              ORDER BY
                    a.DT_UPDATE DESC
                  , a.loaddate DESC
          ) AS rn

    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    INNER JOIN #main_bal b
        ON b.con_id = TRY_CAST(a.CON_ID AS bigint)
)

SELECT
      con_id
    , flag_newmoney
    , flag_other_markup

INTO #main_attr

FROM x
WHERE rn = 1;



/* ============================================================
   ПОДГОТОВКА
   ============================================================ */

IF OBJECT_ID('tempdb..#main_prepared') IS NOT NULL DROP TABLE #main_prepared;

SELECT
      b.*

    , ISNULL(a.flag_newmoney,0) AS flag_newmoney
    , ISNULL(a.flag_other_markup,0) AS flag_other_markup


    /* А: автоматическое промо */
    , CASE
        WHEN
               b.TSEGMENTNAME = N'ДЧБО'
            OR b.PROD_NAME_res IN
               (
                   N'Классический',
                   N'Привилегия',
                   N'Достояние'
               )
        THEN 1 ELSE 0
      END AS is_auto_promo


    /* обычный срок */
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
      END AS normal_term_bucket

INTO #main_prepared

FROM #main_bal b

LEFT JOIN #main_attr a
    ON a.con_id = b.con_id;



/* ============================================================
   МАТЧ К СПРАВОЧНИКУ + ФИНАЛЬНЫЙ ФЛАГ ПРОМО
   ============================================================ */

IF OBJECT_ID('tempdb..#main_result') IS NOT NULL DROP TABLE #main_result;

SELECT
      p.*

    , d.id          AS dict_id
    , d.term_bucket AS dict_term_bucket
    , d.promo_rate  AS dict_rate

    , CASE
        WHEN p.is_auto_promo = 1
            THEN 1

        WHEN p.is_auto_promo = 0
         AND p.flag_newmoney = 1
         AND d.id IS NOT NULL
            THEN 1

        ELSE 0
      END AS is_promo


    /* срок промо:
       если справочник сматчился — его term_bucket;
       для AUTO пытаемся взять активный промо-бакет по дате+сроку;
       иначе обычный mapping
    */
    , COALESCE(
          d.term_bucket,
          db.term_bucket,
          p.normal_term_bucket
      ) AS promo_term_bucket

INTO #main_result

FROM #main_prepared p


/* полный матч справочника */
OUTER APPLY
(
    SELECT TOP (1)
          r.id
        , r.term_bucket
        , r.promo_rate

    FROM ALM_TEST.WORK.promo_new_money_rate_dict r WITH (NOLOCK)

    WHERE
        r.is_active = 1

        AND p.dt_open
            BETWEEN r.date_from AND r.date_to

        AND p.termdays
            BETWEEN r.term_min AND r.term_max

        AND r.conv_type =
            CASE
                WHEN p.conv_norm = 'AT_THE_END'
                    THEN 'AT_THE_END'
                ELSE 'NOT_AT_THE_END'
            END

        AND p.out_rub
            BETWEEN r.amount_from AND r.amount_to

        AND ABS(
                p.rate_con_class - r.promo_rate
            ) <= @eps

    ORDER BY
          r.date_from DESC
        , r.id DESC
) d


/* только для определения названия срока AUTO-промо */
OUTER APPLY
(
    SELECT TOP (1)
        r.term_bucket

    FROM ALM_TEST.WORK.promo_new_money_rate_dict r WITH (NOLOCK)

    WHERE
        r.is_active = 1

        AND p.dt_open
            BETWEEN r.date_from AND r.date_to

        AND p.termdays
            BETWEEN r.term_min AND r.term_max

    ORDER BY
          r.date_from DESC
        , r.id DESC
) db;



/* ============================================================
   RESULT SET №1
   НОВЫЕ ПРИВЛЕЧЕНИЯ

   Промо показываем отдельными строками:
   61 РК / 91 РК / 122 РК и т.д.
   ============================================================ */

;WITH tall AS
(
    SELECT
          CASE
            WHEN is_promo = 1
                THEN CONCAT(promo_term_bucket,N' РК')
            ELSE CAST(normal_term_bucket AS nvarchar(20))
          END AS [Срок, дн.]

        , dt_open AS [Дата открытия]

        , SUM(out_rub) AS [Объем, руб.]

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

        , CASE
            WHEN is_promo = 1
                THEN promo_term_bucket
            ELSE normal_term_bucket
          END AS sort_term

        , is_promo AS sort_promo

    FROM #main_result

    GROUP BY
          dt_open
        , is_promo
        , promo_term_bucket
        , normal_term_bucket
)

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
    , sort_promo;





/* ============================================================
   ============================================================
   ЧАСТЬ 2
   БАЛАНС НА @control_dt
   ============================================================
   ============================================================ */

IF OBJECT_ID('tempdb..#bal_check') IS NOT NULL DROP TABLE #bal_check;

SELECT
      TRY_CAST(t.con_id AS bigint) AS con_id
    , MIN(TRY_CAST(t.cli_id AS bigint)) AS cli_id

    , MIN(CAST(t.dt_open AS date)) AS dt_open

    , MIN(t.TSEGMENTNAME)  AS TSEGMENTNAME
    , MIN(t.PROD_NAME_res) AS PROD_NAME_res

    , SUM(t.out_rub) AS out_rub

    , MIN(t.rate_con) AS rate_con
    , MIN(t.termdays) AS termdays

    , CASE
        WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(t.conv,''))),'')) IS NULL
            THEN 'AT_THE_END'
        ELSE UPPER(MIN(NULLIF(LTRIM(RTRIM(COALESCE(t.conv,''))),'')))
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

IF OBJECT_ID('tempdb..#attr_check') IS NOT NULL DROP TABLE #attr_check;

;WITH x AS
(
    SELECT
          TRY_CAST(a.CON_ID AS bigint) AS con_id

        , CASE
            WHEN
                   ISNULL(TRY_CAST(a.[Нов] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[НДП] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[НДМ] AS int),0) = 1
            THEN 1 ELSE 0
          END AS flag_newmoney

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
            THEN 1 ELSE 0
          END AS flag_other_markup

        , ROW_NUMBER() OVER
          (
              PARTITION BY a.CON_ID
              ORDER BY
                    a.DT_UPDATE DESC
                  , a.loaddate DESC
          ) AS rn

    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    INNER JOIN #bal_check b
        ON b.con_id = TRY_CAST(a.CON_ID AS bigint)
)

SELECT
      con_id
    , flag_newmoney
    , flag_other_markup

INTO #attr_check

FROM x
WHERE rn = 1;



/* ============================================================
   ФИНАЛЬНАЯ КЛАССИФИКАЦИЯ БАЛАНСА
   ============================================================ */

IF OBJECT_ID('tempdb..#result_check') IS NOT NULL DROP TABLE #result_check;

SELECT
      b.*

    , ISNULL(a.flag_newmoney,0)
        AS flag_newmoney

    , ISNULL(a.flag_other_markup,0)
        AS flag_other_markup


    /* A */
    , CASE
        WHEN
               b.TSEGMENTNAME = N'ДЧБО'
            OR b.PROD_NAME_res IN
               (
                   N'Классический',
                   N'Привилегия',
                   N'Достояние'
               )
        THEN 1 ELSE 0
      END AS is_auto_promo


    /* B: полный матч справочника */
    , CASE
        WHEN d.id IS NOT NULL
            THEN 1
        ELSE 0
      END AS dict_match


    /* итоговый промо */
    , CASE

        WHEN
               b.TSEGMENTNAME = N'ДЧБО'
            OR b.PROD_NAME_res IN
               (
                   N'Классический',
                   N'Привилегия',
                   N'Достояние'
               )
            THEN 1

        WHEN
               ISNULL(a.flag_newmoney,0) = 1
           AND d.id IS NOT NULL
            THEN 1

        ELSE 0

      END AS is_promo


    /* информация из справочника */
    , d.id AS dict_id
    , d.term_bucket AS dict_term_bucket
    , d.promo_rate AS dict_promo_rate
    , d.campaign_name

INTO #result_check

FROM #bal_check b

LEFT JOIN #attr_check a
    ON a.con_id = b.con_id

OUTER APPLY
(
    SELECT TOP (1)
          r.id
        , r.term_bucket
        , r.promo_rate
        , r.campaign_name

    FROM ALM_TEST.WORK.promo_new_money_rate_dict r WITH (NOLOCK)

    WHERE
        r.is_active = 1

        AND b.dt_open
            BETWEEN r.date_from AND r.date_to

        AND b.termdays
            BETWEEN r.term_min AND r.term_max

        AND r.conv_type =
            CASE
                WHEN b.conv_norm = 'AT_THE_END'
                    THEN 'AT_THE_END'
                ELSE 'NOT_AT_THE_END'
            END

        AND b.out_rub
            BETWEEN r.amount_from AND r.amount_to

        AND ABS(
                b.rate_con - r.promo_rate
            ) <= @eps

    ORDER BY
          r.date_from DESC
        , r.id DESC
) d;



/* ============================================================
   RESULT SET №2
   КОНТРОЛЬНЫЕ ОБЪЁМЫ НА ДАТУ
   ============================================================ */

SELECT
      SUM(out_rub)
        AS [Весь баланс]


    /* все промо */
    , SUM(
        CASE WHEN is_promo = 1
             THEN out_rub ELSE 0 END
      ) AS [Промо всего]


    /* промо по правилу А */
    , SUM(
        CASE WHEN is_auto_promo = 1
             THEN out_rub ELSE 0 END
      ) AS [Промо AUTO - ДЧБО или продукт]


    /* промо по правилу Б */
    , SUM(
        CASE
            WHEN is_auto_promo = 0
             AND flag_newmoney = 1
             AND dict_match = 1
            THEN out_rub ELSE 0
        END
      ) AS [Промо - НДП НДМ НОВ + справочник]


    /* всё, что НЕ признали промо */
    , SUM(
        CASE WHEN is_promo = 0
             THEN out_rub ELSE 0 END
      ) AS [Не промо всего]


    /* =========================================
       ПОЧЕМУ НЕ ПОПАЛО В ПРОМО
       ========================================= */

    /* есть новые деньги, но промо не признали */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND flag_newmoney = 1
            THEN out_rub ELSE 0
        END
      ) AS [Не промо - есть НДП НДМ НОВ]


    /* конкретно: надбавка есть, справочника нет */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND flag_newmoney = 1
             AND dict_match = 0
            THEN out_rub ELSE 0
        END
      ) AS [Не промо - НДП НДМ НОВ без справочника]


    /* ставка/условия подходят, надбавки новых денег нет */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND flag_newmoney = 0
             AND dict_match = 1
            THEN out_rub ELSE 0
        END
      ) AS [Не промо - справочник без НДП НДМ НОВ]


    /* вообще подходят под справочник */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND dict_match = 1
            THEN out_rub ELSE 0
        END
      ) AS [Не промо - есть матч справочника]


    /* прочие надбавки */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND flag_other_markup = 1
            THEN out_rub ELSE 0
        END
      ) AS [Не промо - прочие надбавки]


    /* ровно интересующая остаточная группа */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND flag_other_markup = 1
             AND flag_newmoney = 0
             AND dict_match = 0
            THEN out_rub ELSE 0
        END
      ) AS [Не промо - прочие надбавки без новых денег и справочника]

FROM #result_check;



/* ============================================================
   RESULT SET №3
   ТО ЖЕ В РАЗБИВКЕ TSEGMENTNAME
   ============================================================ */

SELECT
      ISNULL(TSEGMENTNAME,N'NULL') AS TSEGMENTNAME

    , SUM(out_rub)
        AS [Весь баланс]

    , SUM(CASE
            WHEN is_promo = 1
            THEN out_rub ELSE 0
          END)
        AS [Промо всего]

    , SUM(CASE
            WHEN is_auto_promo = 1
            THEN out_rub ELSE 0
          END)
        AS [Промо AUTO]

    , SUM(CASE
            WHEN is_auto_promo = 0
             AND flag_newmoney = 1
             AND dict_match = 1
            THEN out_rub ELSE 0
          END)
        AS [Промо НДП НДМ НОВ + справочник]

    , SUM(CASE
            WHEN is_promo = 0
            THEN out_rub ELSE 0
          END)
        AS [Не промо]

    , SUM(CASE
            WHEN is_promo = 0
             AND flag_newmoney = 1
             AND dict_match = 0
            THEN out_rub ELSE 0
          END)
        AS [Не промо - новые деньги без справочника]

    , SUM(CASE
            WHEN is_promo = 0
             AND flag_newmoney = 0
             AND dict_match = 1
            THEN out_rub ELSE 0
          END)
        AS [Не промо - справочник без новых денег]

    , SUM(CASE
            WHEN is_promo = 0
             AND flag_other_markup = 1
             AND flag_newmoney = 0
             AND dict_match = 0
            THEN out_rub ELSE 0
          END)
        AS [Не промо - только прочие надбавки]

FROM #result_check

GROUP BY
    TSEGMENTNAME

ORDER BY
    TSEGMENTNAME;



/* ============================================================
   RESULT SET №4
   ПОДОГОВОРНОЕ ПОЛОТНО НА @control_dt
   ============================================================ */

SELECT
      con_id
    , cli_id

    , TSEGMENTNAME
    , PROD_NAME_res

    , dt_open
    , termdays

    , out_rub
    , rate_con
    , conv_norm

    , CASE
        WHEN conv_norm = 'AT_THE_END'
            THEN 'AT_THE_END'
        ELSE 'NOT_AT_THE_END'
      END AS conv_для_справочника

    , is_auto_promo
    , flag_newmoney
    , dict_match
    , flag_other_markup

    , is_promo

    , CASE

        WHEN TSEGMENTNAME = N'ДЧБО'
            THEN N'ПРОМО: ДЧБО'

        WHEN PROD_NAME_res IN
             (
                 N'Классический',
                 N'Привилегия',
                 N'Достояние'
             )
            THEN N'ПРОМО: продукт AUTO'

        WHEN flag_newmoney = 1
         AND dict_match = 1
            THEN N'ПРОМО: НДП/НДМ/НОВ + справочник'

        WHEN flag_newmoney = 1
         AND dict_match = 0
            THEN N'НЕ ПРОМО: новые деньги без справочника'

        WHEN flag_newmoney = 0
         AND dict_match = 1
            THEN N'НЕ ПРОМО: справочник без новых денег'

        WHEN flag_other_markup = 1
            THEN N'НЕ ПРОМО: прочие надбавки'

        ELSE N'НЕ ПРОМО'

      END AS promo_reason

    , dict_id
    , dict_term_bucket
    , dict_promo_rate
    , campaign_name

FROM #result_check

ORDER BY
      is_promo DESC
    , TSEGMENTNAME
    , out_rub DESC;
