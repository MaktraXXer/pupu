USE ALM_TEST;
GO

ALTER FUNCTION [reports].[fn_NewAttractionVolumes]
(
      @ReportDate date
    , @OpenFrom date
    , @OpenTo date
    , @ExcludeMP bit = 1
)
RETURNS TABLE
AS
RETURN
(
    WITH mp_list AS
    (
        SELECT prod_name_res
        FROM (VALUES
              (N'ДОМа надёжно')
            , (N'Всё в ДОМ')
            , (N'Надёжный')
            , (N'Надёжный VIP')
            , (N'Надёжный премиум')
            , (N'Надёжныйпромо')
            , (N'Надёжный промо')
            , (N'Надёжный старт')
            , (N'Надёжный T2')
            , (N'Надёжный Т2')
            , (N'Надёжный Мегафон')
            , (N'Надёжныйпрайм')
            , (N'Надёжный прайм')
            , (N'Надёжный процент')
            , (N'Надёжныйпроцент')
            , (N'Могучий')
        ) m(prod_name_res)
    ),

    /* 1. Source balances */

    src AS
    (
        SELECT
              v.CON_ID
            , v.OUT_RUB
            , v.TSegmentName
            , v.PROD_NAME_res
            , v.termdays
            , v.rate_con
            , v.conv
            , CAST(v.dt_open AS date) AS dt_open
            , CASE WHEN mp.prod_name_res IS NOT NULL
                   THEN 1 ELSE 0 END AS is_mp
            , CASE
                WHEN v.TSegmentName = N'ДЧБО'
                  OR v.PROD_NAME_res IN
                     (
                         N'Классический',
                         N'Привилегия',
                         N'Достояние'
                     )
                THEN 1 ELSE 0
              END AS is_private

        FROM ALM.ALM.vw_balance_rest_all v WITH (NOLOCK)

        LEFT JOIN mp_list mp
            ON mp.prod_name_res = v.PROD_NAME_res

        WHERE v.DT_REP = @ReportDate
          AND v.dt_open >= @OpenFrom
          AND v.dt_open < DATEADD(day, 1, @OpenTo)

          AND v.section_name = N'Срочные'
          AND v.block_name = N'Привлечение ФЛ'
          AND v.ap = N'Пассив'
          AND v.od_flag = 1
          AND v.cur = '810'
          AND v.OUT_RUB > 0
          AND ISNULL(v.is_floatrate,0) = 0

          AND v.termdays BETWEEN 28 AND 1830

          AND NOT EXISTS
          (
              SELECT 1
              FROM LIQUIDITY.liq.man_FloatContracts fc WITH (NOLOCK)
              WHERE fc.CON_ID = v.CON_ID
          )

          AND (@ExcludeMP = 0 OR mp.prod_name_res IS NULL)
    ),

    /* 2. Latest markups for relevant contracts */

    attr_ranked AS
    (
        SELECT
              a.CON_ID
            , a.[Нов]
            , a.[НДП]
            , a.[НДМ]
            , a.[Мпл]
            , a.[Пнс]
            , a.[От1]
            , a.[Пк2]

            , ROW_NUMBER() OVER
              (
                  PARTITION BY a.CON_ID
                  ORDER BY a.DT_UPDATE DESC, a.loaddate DESC
              ) AS rn

        FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

        INNER JOIN
        (
            SELECT DISTINCT CON_ID
            FROM src
        ) s
            ON s.CON_ID = a.CON_ID
    ),

    /* 3. Dictionary and markup flags */

    matched AS
    (
        SELECT
              s.*
            , d.id AS dict_id

            , CASE
                WHEN ISNULL(TRY_CAST(a.[Нов] AS int),0) = 1
                  OR ISNULL(TRY_CAST(a.[НДП] AS int),0) = 1
                  OR ISNULL(TRY_CAST(a.[НДМ] AS int),0) = 1
                  OR ISNULL(TRY_CAST(a.[Мпл] AS int),0) = 1
                  OR ISNULL(TRY_CAST(a.[Пнс] AS int),0) = 1
                THEN 1 ELSE 0
              END AS new_money

            , CASE
                WHEN ISNULL(TRY_CAST(a.[От1] AS int),0) = 1
                  OR ISNULL(TRY_CAST(a.[Пк2] AS int),0) = 1
                THEN 1 ELSE 0
              END AS retention

        FROM src s

        LEFT JOIN attr_ranked a
            ON a.CON_ID = s.CON_ID
           AND a.rn = 1

        LEFT JOIN ALM_TEST.WORK.promo_new_money_rate_dict d
            ON s.dt_open BETWEEN d.date_from AND d.date_to
           AND s.termdays BETWEEN d.term_min AND d.term_max

           AND d.conv_type =
               CASE
                   WHEN ISNULL(s.conv,'AT_THE_END') = 'AT_THE_END'
                       THEN 'AT_THE_END'
                   ELSE 'NOT_AT_THE_END'
               END

           AND s.OUT_RUB BETWEEN d.amount_from AND d.amount_to
           AND ABS(s.rate_con - d.promo_rate) <= 0.0005
    ),

    /* 4. Categories and validation */

    classified AS
    (
        SELECT
              m.*

            , CASE
                WHEN m.is_mp = 1
                    THEN N'вклады маркетплейса'

                WHEN m.is_private = 1
                    THEN N'вклад private-сегмента'

                WHEN m.dict_id IS NOT NULL
                 AND m.new_money = 1
                    THEN N'вклад масс-сегмента с надбавкой равной новым деньгам'

                WHEN m.retention = 1
                    THEN N'вклад масс-сегмента на удержание'

                WHEN m.PROD_NAME_res = N'Мой дом без опций'
                    THEN N'вклад масс-сегмента базовый'

                ELSE N'нетипичные вклады масс-сегмента'
              END AS segment_name

            /* Error codes contain the contract ID */

            , CASE
                WHEN m.is_mp = 0 AND m.is_private = 0
                 AND m.new_money = 1 AND m.dict_id IS NULL
                    THEN CONCAT('ERR_NEW_MONEY_NO_DICT_CON_ID=',m.CON_ID)

                WHEN m.is_mp = 0 AND m.is_private = 0
                 AND m.dict_id IS NOT NULL AND m.new_money = 0
                    THEN CONCAT('ERR_DICT_NO_NEW_MONEY_CON_ID=',m.CON_ID)

                WHEN m.is_mp = 0 AND m.is_private = 0
                 AND m.dict_id IS NOT NULL
                 AND m.new_money = 1 AND m.retention = 1
                    THEN CONCAT('ERR_NEW_MONEY_AND_RETENTION_CON_ID=',m.CON_ID)

                ELSE '0'
              END AS validation_code

            , CASE
                WHEN termdays BETWEEN 28 AND 44 THEN 31
                WHEN termdays BETWEEN 45 AND 70 THEN 61
                WHEN termdays BETWEEN 85 AND 119 THEN 91
                WHEN termdays BETWEEN 120 AND 140 THEN 124
                WHEN termdays BETWEEN 141 AND 170 THEN 151
                WHEN termdays BETWEEN 175 AND 200 THEN 181
                WHEN termdays BETWEEN 201 AND 230 THEN 212
                WHEN termdays BETWEEN 231 AND 250 THEN 243
                WHEN termdays BETWEEN 251 AND 290 THEN 274
                WHEN termdays BETWEEN 340 AND 405 THEN 365
                WHEN termdays BETWEEN 540 AND 621 THEN 550
                WHEN termdays BETWEEN 720 AND 763 THEN 750
                WHEN termdays BETWEEN 1090 AND 1140 THEN 1100
                WHEN termdays BETWEEN 1450 AND 1475 THEN 1460
                WHEN termdays BETWEEN 1795 AND 1830 THEN 1825
                ELSE termdays
              END AS bucket_code

        FROM matched m
    ),

    /* 5. Aggregation by category and term bucket */

    seg_bucket AS
    (
        SELECT
              bucket_code
            , segment_name

            , SUM(OUT_RUB)
              + SUM(CONVERT(int, validation_code)) AS segment_vol

        FROM classified
        GROUP BY bucket_code, segment_name
    ),

    seg_total AS
    (
        SELECT
              segment_name
            , SUM(segment_vol) AS segment_total
        FROM seg_bucket
        GROUP BY segment_name
    ),

    bucket_total AS
    (
        SELECT
              bucket_code
            , SUM(segment_vol) AS bucket_total
        FROM seg_bucket
        GROUP BY bucket_code
    ),

    grand AS
    (
        SELECT SUM(bucket_total) AS grand_total
        FROM bucket_total
    )

    /* 6. Final output */

    SELECT
          sb.bucket_code AS Bucket
        , sb.segment_name AS Segment
        , sb.segment_vol AS SegmentVolume

        , CAST(
              sb.segment_vol / NULLIF(st.segment_total,0)
              AS decimal(18,4)
          ) AS SegmentSharePct

        , bt.bucket_total AS BucketVolume

        , CAST(
              bt.bucket_total / NULLIF(g.grand_total,0)
              AS decimal(18,4)
          ) AS BucketSharePct

    FROM seg_bucket sb

    JOIN seg_total st
        ON st.segment_name = sb.segment_name

    JOIN bucket_total bt
        ON bt.bucket_code = sb.bucket_code

    CROSS JOIN grand g

    UNION ALL

    SELECT
          bt.bucket_code
        , N'Итого'
        , bt.bucket_total

        , CAST(
              bt.bucket_total / NULLIF(g.grand_total,0)
              AS decimal(18,4)
          )

        , bt.bucket_total

        , CAST(
              bt.bucket_total / NULLIF(g.grand_total,0)
              AS decimal(18,4)
          )

    FROM bucket_total bt
    CROSS JOIN grand g
);
GO
