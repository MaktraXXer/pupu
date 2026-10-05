USE ALM;
SET NOCOUNT ON;

DECLARE @dt_rep      date = '2026-10-03';
DECLARE @date_from   date = '2026-10-01';
DECLARE @date_to     date = '2026-10-03';

DECLARE @control_dt  date = '2026-09-30';

DECLARE @eps decimal(9,6) = 0.0005;


/* ============================================================
   1. НОВЫЕ ПРИВЛЕЧЕНИЯ
   ============================================================ */

IF OBJECT_ID('tempdb..#new') IS NOT NULL DROP TABLE #new;

SELECT
      t.dt_open
    , t.con_id
    , t.cli_id
    , t.TSEGMENTNAME
    , t.PROD_NAME_RES

    , t.out_rub
    , t.rate_con
    , t.rate_trf
    , t.termdays
    , fk.AVG_KEY_RATE

    , CASE
        WHEN t.termdays BETWEEN 28   AND 44   THEN 31
        WHEN t.termdays BETWEEN 45   AND 79   THEN 61
        WHEN t.termdays BETWEEN 80   AND 115  THEN 91
        WHEN t.termdays BETWEEN 116  AND 140  THEN 124
        WHEN t.termdays BETWEEN 141  AND 174  THEN 151
        WHEN t.termdays BETWEEN 175  AND 200  THEN 181
        WHEN t.termdays BETWEEN 201  AND 230  THEN 212
        WHEN t.termdays BETWEEN 231  AND 250  THEN 243
        WHEN t.termdays BETWEEN 251  AND 290  THEN 274
        WHEN t.termdays BETWEEN 340  AND 405  THEN 365
        WHEN t.termdays BETWEEN 540  AND 621  THEN 550
        WHEN t.termdays BETWEEN 720  AND 763  THEN 750
        WHEN t.termdays BETWEEN 1090 AND 1140 THEN 1100
        WHEN t.termdays BETWEEN 1450 AND 1475 THEN 1460
        WHEN t.termdays BETWEEN 1795 AND 1830 THEN 1825
        ELSE t.termdays
      END AS normal_bucket

    , d.term_bucket AS promo_bucket

    , CASE
        /* справочник не совпал -> не промо */
        WHEN d.term_bucket IS NULL THEN 0

        /* A:
           ДЧБО ИЛИ специальные продукты.
           Достаточно совпадения со справочником */
        WHEN t.TSEGMENTNAME = N'ДЧБО'
          OR t.PROD_NAME_RES IN
             (N'Классический', N'Привилегия', N'Достояние')
            THEN 1

        /* B:
           все остальные.
           Нужен НОВ/НДП/НДМ и НЕ должно быть других надбавок */
        WHEN ISNULL(a.new_money_flag,0) = 1
         AND ISNULL(a.other_markup_flag,0) = 0
            THEN 1

        ELSE 0
      END AS is_promo

INTO #new

FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)

LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fk
    ON fk.DT_REP = t.dt_open
   AND fk.TERM   = t.termdays


/* последняя запись надбавок */
OUTER APPLY
(
    SELECT TOP (1)

        CASE
            WHEN
                   ISNULL(TRY_CAST(x.[Нов] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[НДП] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[НДМ] AS int),0) = 1
            THEN 1 ELSE 0
        END AS new_money_flag,

        CASE
            WHEN
                   ISNULL(TRY_CAST(x.[Пк3] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пк6] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пк7] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пр2] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пр3] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пк2] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[От1] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Мпл] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пнс] AS int),0) = 1
            THEN 1 ELSE 0
        END AS other_markup_flag

    FROM ALM.ehd.attr_DepoFLConditions x WITH (NOLOCK)

    WHERE TRY_CAST(x.CON_ID AS bigint) = t.con_id

    ORDER BY
          x.DT_UPDATE DESC
        , x.loaddate DESC
) a


/* совпадение со справочником промо */
OUTER APPLY
(
    SELECT TOP (1)
        r.term_bucket

    FROM ALM_TEST.WORK.promo_new_money_rate_dict r WITH (NOLOCK)

    WHERE
        r.is_active = 1

        AND t.dt_open
            BETWEEN r.date_from AND r.date_to

        AND t.termdays
            BETWEEN r.term_min AND r.term_max

        /* AT_THE_END отдельно,
           любая другая conv -> NOT_AT_THE_END */
        AND r.conv_type =
            CASE
                WHEN ISNULL(t.conv,'AT_THE_END') = 'AT_THE_END'
                    THEN 'AT_THE_END'
                ELSE 'NOT_AT_THE_END'
            END

        AND t.out_rub
            BETWEEN r.amount_from AND r.amount_to

        AND ABS(t.rate_con - r.promo_rate) <= @eps

    ORDER BY r.id DESC
) d


WHERE
    t.dt_rep = @dt_rep
    AND t.dt_open BETWEEN @date_from AND @date_to

    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.od_flag      = 1
    AND t.cur          = '810'

    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;



/* ============================================================
   RESULT SET 1
   НОВЫЕ ПРИВЛЕЧЕНИЯ
   ============================================================ */

SELECT
      CASE
          WHEN is_promo = 1
              THEN CONCAT(promo_bucket, N' РК')
          ELSE CAST(normal_bucket AS nvarchar(20))
      END AS [Срок, дн.]

    , CAST(dt_open AS date) AS [Дата открытия]

    , SUM(out_rub) AS [Объем, руб.]

    , CAST(
        SUM(out_rub * rate_con)
        / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END),0)
        AS decimal(9,6)
      ) AS [Средневзв. ставка (клиент)]

    , CAST(
        SUM(out_rub * rate_trf)
        / NULLIF(SUM(CASE WHEN rate_trf IS NOT NULL THEN out_rub END),0)
        AS decimal(9,6)
      ) AS [Средневзв. ставка (ТС)]

    , CAST(
        SUM(out_rub * AVG_KEY_RATE)
        / NULLIF(SUM(CASE WHEN AVG_KEY_RATE IS NOT NULL THEN out_rub END),0)
        AS decimal(9,6)
      ) AS [Средневзв. прогнозный КС]

    , CAST(
        SUM(out_rub * termdays)
        / NULLIF(SUM(out_rub),0)
        AS decimal(18,2)
      ) AS [Средневзв. контрактная срочность]

FROM #new

GROUP BY
      CAST(dt_open AS date)
    , CASE
          WHEN is_promo = 1
              THEN CONCAT(promo_bucket, N' РК')
          ELSE CAST(normal_bucket AS nvarchar(20))
      END

ORDER BY
      [Дата открытия]
    , TRY_CONVERT(
          int,
          REPLACE([Срок, дн.],N' РК','')
      );



/* ============================================================
   2. БАЛАНС НА КОНТРОЛЬНУЮ ДАТУ
   ============================================================ */

IF OBJECT_ID('tempdb..#control') IS NOT NULL DROP TABLE #control;

SELECT
      t.con_id
    , t.cli_id
    , t.TSEGMENTNAME
    , t.PROD_NAME_RES
    , t.dt_open

    , t.out_rub
    , t.rate_con
    , t.termdays

    , ISNULL(a.new_money_flag,0) AS new_money_flag
    , ISNULL(a.other_markup_flag,0) AS other_markup_flag

    , CASE
        WHEN d.term_bucket IS NOT NULL THEN 1
        ELSE 0
      END AS dict_match


    /* то же самое итоговое правило A/B */
    , CASE

        WHEN d.term_bucket IS NULL
            THEN 0

        /* A */
        WHEN t.TSEGMENTNAME = N'ДЧБО'
          OR t.PROD_NAME_RES IN
             (N'Классический', N'Привилегия', N'Достояние')
            THEN 1

        /* B */
        WHEN ISNULL(a.new_money_flag,0) = 1
         AND ISNULL(a.other_markup_flag,0) = 0
            THEN 1

        ELSE 0

      END AS is_promo

INTO #control

FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)


OUTER APPLY
(
    SELECT TOP (1)

        CASE
            WHEN
                   ISNULL(TRY_CAST(x.[Нов] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[НДП] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[НДМ] AS int),0) = 1
            THEN 1 ELSE 0
        END AS new_money_flag,

        CASE
            WHEN
                   ISNULL(TRY_CAST(x.[Пк3] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пк6] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пк7] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пр2] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пр3] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пк2] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[От1] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Мпл] AS int),0) = 1
                OR ISNULL(TRY_CAST(x.[Пнс] AS int),0) = 1
            THEN 1 ELSE 0
        END AS other_markup_flag

    FROM ALM.ehd.attr_DepoFLConditions x WITH (NOLOCK)

    WHERE TRY_CAST(x.CON_ID AS bigint) = t.con_id

    ORDER BY
          x.DT_UPDATE DESC
        , x.loaddate DESC
) a


OUTER APPLY
(
    SELECT TOP (1)
        r.term_bucket

    FROM ALM_TEST.WORK.promo_new_money_rate_dict r WITH (NOLOCK)

    WHERE
        r.is_active = 1

        AND t.dt_open
            BETWEEN r.date_from AND r.date_to

        AND t.termdays
            BETWEEN r.term_min AND r.term_max

        AND r.conv_type =
            CASE
                WHEN ISNULL(t.conv,'AT_THE_END') = 'AT_THE_END'
                    THEN 'AT_THE_END'
                ELSE 'NOT_AT_THE_END'
            END

        AND t.out_rub
            BETWEEN r.amount_from AND r.amount_to

        AND ABS(t.rate_con - r.promo_rate) <= @eps

    ORDER BY r.id DESC
) d


WHERE
    t.dt_rep = @control_dt

    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.od_flag      = 1
    AND t.cur          = '810'

    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;



/* ============================================================
   RESULT SET 2
   ПРОСТАЯ ПРОВЕРКА БАЛАНСА
   ============================================================ */

SELECT
      SUM(out_rub)
        AS [Все вклады]

    /* что реально считаем промо */
    , SUM(
        CASE WHEN is_promo = 1
             THEN out_rub ELSE 0 END
      ) AS [Промо A+B]


    /* написано НОВ/НДП/НДМ,
       но по итоговой логике не попало */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND new_money_flag = 1
            THEN out_rub
            ELSE 0
        END
      ) AS [Не промо, но НОВ НДП НДМ]


    /* полностью совпадает со справочником,
       но итогом не промо */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND dict_match = 1
            THEN out_rub
            ELSE 0
        END
      ) AS [Не промо, но ставка и срок из справочника]


    /* совпадает со справочником,
       но имеются запрещённые другие надбавки */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND dict_match = 1
             AND other_markup_flag = 1
            THEN out_rub
            ELSE 0
        END
      ) AS [Не промо, справочник + другие надбавки]


    /* вообще все договоры с другими надбавками */
    , SUM(
        CASE
            WHEN other_markup_flag = 1
            THEN out_rub
            ELSE 0
        END
      ) AS [Все с другими надбавками]

FROM #control;
