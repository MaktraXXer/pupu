USE ALM;
SET NOCOUNT ON;

DECLARE @dt_rep      date = '2026-10-03';
DECLARE @date_from   date = '2026-10-01';
DECLARE @date_to     date = '2026-10-03';

DECLARE @control_dt  date = '2026-09-30';

DECLARE @eps decimal(9,6) = 0.0005;


/* ============================================================
   1. НОВЫЕ ПРИВЛЕЧЕНИЯ
   con_id уже уникален
   ============================================================ */

IF OBJECT_ID('tempdb..#new_bal') IS NOT NULL DROP TABLE #new_bal;

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
    , t.conv
    , fk.AVG_KEY_RATE

INTO #new_bal

FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)

LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fk
    ON fk.DT_REP = t.dt_open
   AND fk.TERM   = t.termdays

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
   2. БАЛАНС НА КОНТРОЛЬНУЮ ДАТУ
   ============================================================ */

IF OBJECT_ID('tempdb..#control_bal') IS NOT NULL DROP TABLE #control_bal;

SELECT
      t.dt_open
    , t.con_id
    , t.cli_id
    , t.TSEGMENTNAME
    , t.PROD_NAME_RES
    , t.out_rub
    , t.rate_con
    , t.termdays
    , t.conv

INTO #control_bal

FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)

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
   3. ВСЕ НУЖНЫЕ CON_ID
   ============================================================ */

IF OBJECT_ID('tempdb..#con_ids') IS NOT NULL DROP TABLE #con_ids;

SELECT con_id
INTO #con_ids
FROM
(
    SELECT con_id FROM #new_bal
    UNION
    SELECT con_id FROM #control_bal
) x;

CREATE UNIQUE CLUSTERED INDEX IX_con_ids
    ON #con_ids(con_id);



/* ============================================================
   4. НАДБАВКИ — ОДИН РАЗ ДЛЯ ВСЕХ НУЖНЫХ ДОГОВОРОВ

   new_money:
       Нов / НДП / НДМ

   other_markup:
       Пк3 / Пк6 / Пк7
       Пр2 / Пр3
       Пк2 / От1
       Мпл / Пнс
   ============================================================ */

IF OBJECT_ID('tempdb..#attr') IS NOT NULL DROP TABLE #attr;

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
          END AS new_money_flag

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
          END AS other_markup_flag

        , ROW_NUMBER() OVER
          (
              PARTITION BY a.CON_ID
              ORDER BY
                    a.DT_UPDATE DESC
                  , a.loaddate DESC
          ) AS rn

    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    INNER JOIN #con_ids c
        ON c.con_id = TRY_CAST(a.CON_ID AS bigint)
)

SELECT
      con_id
    , new_money_flag
    , other_markup_flag

INTO #attr

FROM x

WHERE rn = 1;

CREATE UNIQUE CLUSTERED INDEX IX_attr
    ON #attr(con_id);



/* ============================================================
   5. НОВЫЕ ПРИВЛЕЧЕНИЯ — КЛАССИФИКАЦИЯ
   ============================================================ */

IF OBJECT_ID('tempdb..#new') IS NOT NULL DROP TABLE #new;

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
      END AS normal_bucket

    , d.term_bucket AS promo_bucket


    /* ========================================================
       A:
       ДЧБО ИЛИ Классический/Привилегия/Достояние
       -> достаточно совпадения со справочником

       B:
       все остальные
       -> справочник + НОВ/НДП/НДМ + нет других надбавок
       ======================================================== */

    , CASE
        WHEN d.term_bucket IS NULL
            THEN 0

        WHEN b.TSEGMENTNAME = N'ДЧБО'
          OR b.PROD_NAME_RES IN
             (N'Классический',N'Привилегия',N'Достояние')
            THEN 1

        WHEN ISNULL(a.new_money_flag,0) = 1
         AND ISNULL(a.other_markup_flag,0) = 0
            THEN 1

        ELSE 0
      END AS is_promo

INTO #new

FROM #new_bal b

LEFT JOIN #attr a
    ON a.con_id = b.con_id

OUTER APPLY
(
    SELECT TOP (1)
        r.term_bucket

    FROM ALM_TEST.WORK.promo_new_money_rate_dict r WITH (NOLOCK)

    WHERE
        r.is_active = 1

        AND b.dt_open
            BETWEEN r.date_from AND r.date_to

        AND b.termdays
            BETWEEN r.term_min AND r.term_max

        AND r.conv_type =
            CASE
                WHEN ISNULL(b.conv,'AT_THE_END') = 'AT_THE_END'
                    THEN 'AT_THE_END'
                ELSE 'NOT_AT_THE_END'
            END

        AND b.out_rub
            BETWEEN r.amount_from AND r.amount_to

        AND ABS(b.rate_con - r.promo_rate) <= @eps

    ORDER BY r.id DESC
) d;



/* ============================================================
   RESULT SET 1
   НОВЫЕ ПРИВЛЕЧЕНИЯ
   ============================================================ */

SELECT
      CASE
          WHEN is_promo = 1
              THEN CONCAT(promo_bucket,N' РК')
          ELSE CAST(normal_bucket AS nvarchar(20))
      END AS [Срок, дн.]

    , CAST(dt_open AS date) AS [Дата открытия]

    , SUM(out_rub) AS [Объем, руб.]

    , CAST(
        SUM(out_rub * rate_con)
        / NULLIF(
            SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END),
            0
        )
        AS decimal(9,6)
      ) AS [Средневзв. ставка (клиент)]

    , CAST(
        SUM(out_rub * rate_trf)
        / NULLIF(
            SUM(CASE WHEN rate_trf IS NOT NULL THEN out_rub END),
            0
        )
        AS decimal(9,6)
      ) AS [Средневзв. ставка (ТС)]

    , CAST(
        SUM(out_rub * AVG_KEY_RATE)
        / NULLIF(
            SUM(CASE WHEN AVG_KEY_RATE IS NOT NULL THEN out_rub END),
            0
        )
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
              THEN CONCAT(promo_bucket,N' РК')
          ELSE CAST(normal_bucket AS nvarchar(20))
      END

ORDER BY
      [Дата открытия]
    , TRY_CONVERT(
          int,
          REPLACE([Срок, дн.],N' РК','')
      );



/* ============================================================
   6. КОНТРОЛЬНЫЙ БАЛАНС
   ============================================================ */

IF OBJECT_ID('tempdb..#control') IS NOT NULL DROP TABLE #control;

SELECT
      b.*

    , ISNULL(a.new_money_flag,0)
        AS new_money_flag

    , ISNULL(a.other_markup_flag,0)
        AS other_markup_flag

    , CASE
        WHEN d.term_bucket IS NOT NULL THEN 1
        ELSE 0
      END AS dict_match


    , CASE
        WHEN d.term_bucket IS NULL
            THEN 0

        WHEN b.TSEGMENTNAME = N'ДЧБО'
          OR b.PROD_NAME_RES IN
             (N'Классический',N'Привилегия',N'Достояние')
            THEN 1

        WHEN ISNULL(a.new_money_flag,0) = 1
         AND ISNULL(a.other_markup_flag,0) = 0
            THEN 1

        ELSE 0
      END AS is_promo

INTO #control

FROM #control_bal b

LEFT JOIN #attr a
    ON a.con_id = b.con_id

OUTER APPLY
(
    SELECT TOP (1)
        r.term_bucket

    FROM ALM_TEST.WORK.promo_new_money_rate_dict r WITH (NOLOCK)

    WHERE
        r.is_active = 1

        AND b.dt_open
            BETWEEN r.date_from AND r.date_to

        AND b.termdays
            BETWEEN r.term_min AND r.term_max

        AND r.conv_type =
            CASE
                WHEN ISNULL(b.conv,'AT_THE_END') = 'AT_THE_END'
                    THEN 'AT_THE_END'
                ELSE 'NOT_AT_THE_END'
            END

        AND b.out_rub
            BETWEEN r.amount_from AND r.amount_to

        AND ABS(b.rate_con - r.promo_rate) <= @eps

    ORDER BY r.id DESC
) d;



/* ============================================================
   RESULT SET 2
   ПРОВЕРКА БАЛАНСА
   ============================================================ */

SELECT
      SUM(out_rub)
        AS [Все вклады]

    , SUM(
        CASE
            WHEN is_promo = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [Промо A+B]

    , SUM(
        CASE
            WHEN is_promo = 0
             AND new_money_flag = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [Не промо, но НОВ НДП НДМ]

    , SUM(
        CASE
            WHEN is_promo = 0
             AND dict_match = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [Не промо, но совпал справочник]

    , SUM(
        CASE
            WHEN is_promo = 0
             AND dict_match = 1
             AND other_markup_flag = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [Не промо, справочник + другие надбавки]

    , SUM(
        CASE
            WHEN other_markup_flag = 1
                THEN out_rub
            ELSE 0
        END
      ) AS [Все с другими надбавками]

FROM #control;
