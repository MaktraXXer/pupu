USE ALM;
SET NOCOUNT ON;

DECLARE @dt_rep date = '2026-09-30';
DECLARE @eps decimal(9,6) = 0.0005;


/* ============================================================
   LAST ATTRIBUTES
   ============================================================ */

;WITH attr AS
(
    SELECT
          a.*
        , ROW_NUMBER() OVER
          (
              PARTITION BY a.CON_ID
              ORDER BY
                    a.DT_UPDATE DESC
                  , a.loaddate DESC
          ) AS rn

    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)
)

SELECT
      t.*

    /* dictionary */
    , d.id            AS dict_id
    , d.term_bucket   AS dict_term_bucket
    , d.conv_type     AS dict_conv_type
    , d.promo_rate    AS dict_promo_rate
    , d.amount_from   AS dict_amount_from
    , d.amount_to     AS dict_amount_to
    , d.date_from     AS dict_date_from
    , d.date_to       AS dict_date_to
    , d.campaign_name AS dict_campaign_name

    /* new money */
    , a.[Нов]
    , a.[НДП]
    , a.[НДМ]

    /* other markups */
    , a.[Пр2]
    , a.[Пр3]

    , a.[Пк2]
    , a.[Пк3]
    , a.[Пк6]
    , a.[Пк7]

    , a.[От1]
    , a.[Мпл]
    , a.[Пнс]

FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)


/* latest markup record */
LEFT JOIN attr a
    ON TRY_CAST(a.CON_ID AS bigint) = t.con_id
   AND a.rn = 1


/* promo dictionary */
LEFT JOIN ALM_TEST.WORK.promo_new_money_rate_dict d WITH (NOLOCK)

    ON d.is_active = 1

   AND CAST(t.dt_open AS date)
       BETWEEN d.date_from AND d.date_to

   AND t.termdays
       BETWEEN d.term_min AND d.term_max

   AND d.conv_type =
       CASE
           WHEN ISNULL(t.conv,'AT_THE_END') = 'AT_THE_END'
               THEN 'AT_THE_END'
           ELSE 'NOT_AT_THE_END'
       END

   AND t.out_rub
       BETWEEN d.amount_from AND d.amount_to

   AND ABS(t.rate_con - d.promo_rate) <= @eps


WHERE
    t.dt_rep = @dt_rep

    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.od_flag      = 1
    AND t.cur          = '810'

    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0


    /* must match promo dictionary */
    AND d.id IS NOT NULL


    /* NO other markups */
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
