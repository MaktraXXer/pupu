USE ALM;
SET NOCOUNT ON;

DECLARE @dt_rep date = '2026-09-30';
DECLARE @eps decimal(9,6) = 0.0005;


SELECT
      t.*

    /* =========================
       PROMO DICTIONARY
       ========================= */
    , d.id              AS dict_id
    , d.term_bucket     AS dict_term_bucket
    , d.conv_type       AS dict_conv_type
    , d.promo_rate      AS dict_promo_rate
    , d.amount_from     AS dict_amount_from
    , d.amount_to       AS dict_amount_to
    , d.date_from       AS dict_date_from
    , d.date_to         AS dict_date_to
    , d.campaign_name   AS dict_campaign_name


    /* =========================
       NEW MONEY MARKUPS
       ========================= */
    , a.[Нов]
    , a.[НДП]
    , a.[НДМ]


    /* =========================
       OTHER MARKUPS
       ========================= */
    , a.[Пр2]
    , a.[Пр3]

    , a.[Пк2]
    , a.[Пк3]
    , a.[Пк6]
    , a.[Пк7]

    , a.[От1]
    , a.[Мпл]
    , a.[Пнс]


    /* convenient check */
    , CASE
        WHEN
               ISNULL(TRY_CAST(a.[Нов] AS int),0) = 1
            OR ISNULL(TRY_CAST(a.[НДП] AS int),0) = 1
            OR ISNULL(TRY_CAST(a.[НДМ] AS int),0) = 1
        THEN 1
        ELSE 0
      END AS new_money_flag


FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)


/* ============================================================
   LAST ATTRIBUTE RECORD
   ============================================================ */

OUTER APPLY
(
    SELECT TOP (1)
          x.[Нов]
        , x.[НДП]
        , x.[НДМ]

        , x.[Пр2]
        , x.[Пр3]

        , x.[Пк2]
        , x.[Пк3]
        , x.[Пк6]
        , x.[Пк7]

        , x.[От1]
        , x.[Мпл]
        , x.[Пнс]

    FROM ALM.ehd.attr_DepoFLConditions x WITH (NOLOCK)

    WHERE TRY_CAST(x.CON_ID AS bigint) = t.con_id

    ORDER BY
          x.DT_UPDATE DESC
        , x.loaddate DESC
) a


/* ============================================================
   PROMO DICTIONARY MATCH
   ============================================================ */

CROSS APPLY
(
    SELECT TOP (1)
          r.id
        , r.term_bucket
        , r.conv_type
        , r.promo_rate
        , r.amount_from
        , r.amount_to
        , r.date_from
        , r.date_to
        , r.campaign_name

    FROM ALM_TEST.WORK.promo_new_money_rate_dict r WITH (NOLOCK)

    WHERE
        r.is_active = 1

        /* opening date */
        AND CAST(t.dt_open AS date)
            BETWEEN r.date_from AND r.date_to

        /* contractual term */
        AND t.termdays
            BETWEEN r.term_min AND r.term_max

        /* convention:
           AT_THE_END -> AT_THE_END
           everything else -> NOT_AT_THE_END */
        AND r.conv_type =
            CASE
                WHEN ISNULL(t.conv,'AT_THE_END') = 'AT_THE_END'
                    THEN 'AT_THE_END'
                ELSE 'NOT_AT_THE_END'
            END

        /* amount */
        AND t.out_rub
            BETWEEN r.amount_from AND r.amount_to

        /* rate */
        AND ABS(t.rate_con - r.promo_rate) <= @eps

    ORDER BY r.id DESC
) d


WHERE
    t.dt_rep = @dt_rep

    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.od_flag      = 1
    AND t.cur          = '810'

    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0


    /* ========================================================
       NO OTHER MARKUPS
       Нов / НДП / НДМ ARE ALLOWED
       ======================================================== */

    AND ISNULL(TRY_CAST(a.[Пр2] AS int),0) = 0
    AND ISNULL(TRY_CAST(a.[Пр3] AS int),0) = 0

    AND ISNULL(TRY_CAST(a.[Пк2] AS int),0) = 0
    AND ISNULL(TRY_CAST(a.[Пк3] AS int),0) = 0
    AND ISNULL(TRY_CAST(a.[Пк6] AS int),0) = 0
    AND ISNULL(TRY_CAST(a.[Пк7] AS int),0) = 0

    AND ISNULL(TRY_CAST(a.[От1] AS int),0) = 0
    AND ISNULL(TRY_CAST(a.[Мпл] AS int),0) = 0
    AND ISNULL(TRY_CAST(a.[Пнс] AS int),0) = 0


ORDER BY
      t.dt_open
    , t.out_rub DESC;
