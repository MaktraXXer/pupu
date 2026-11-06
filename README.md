
USE [ALM_TEST];
GO

SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

ALTER FUNCTION [reports].[fn_NewAttractionVolumes_forDimaMASS]
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
        SELECT DISTINCT prod_name_res
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

            , CASE
                WHEN mp.prod_name_res IS NOT NULL THEN 1
                ELSE 0
              END AS is_mp

            , CASE
                WHEN v.termdays BETWEEN 45 AND 70
                 AND ISNULL(v.conv,'AT_THE_END') = 'AT_THE_END'
                 AND ABS(v.rate_con - 0.1500) <= 0.0005
                    THEN 1
                ELSE 0
              END AS special_2m

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
          AND v.rate_con IS NOT NULL
          AND v.termdays BETWEEN 28 AND 1830

          AND NOT EXISTS
          (
              SELECT 1
              FROM LIQUIDITY.liq.man_FloatContracts fc WITH (NOLOCK)
              WHERE fc.CON_ID = v.CON_ID
          )

          AND (
              @ExcludeMP = 0
              OR mp.prod_name_res IS NULL
          )

          AND (
              v.TSegmentName IS NULL
              OR v.TSegmentName <> N'ДЧБО'
          )

          AND (
              v.PROD_NAME_res IS NULL
              OR v.PROD_NAME_res NOT IN
              (
                  N'Классический',
                  N'Привилегия',
                  N'Достояние',
                  N'Достояние без опций',
                  N'Достояние с пополнением и снятием',
                  N'Достояние (перем.ст.) без опций',
                  N'Достояние (перем.ст.) с пополнением и снятием'
              )
          )
    ),

    joined AS
    (
        SELECT
              s.*
            , d.dict_match
            , a.[Нов]
            , a.[НДП]
            , a.[НДМ]
            , a.[Мпл]
            , a.[Пнс]
            , a.[От1]
            , a.[Пк2]

        FROM src s

        OUTER APPLY
        (
            SELECT TOP (1)
                  a.[Нов]
                , a.[НДП]
                , a.[НДМ]
                , a.[Мпл]
                , a.[Пнс]
                , a.[От1]
                , a.[Пк2]

            FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

            WHERE a.CON_ID = s.CON_ID
              AND s.is_mp = 0

            ORDER BY
                  a.DT_UPDATE DESC
                , a.loaddate DESC
        ) a

        OUTER APPLY
        (
            SELECT TOP (1)
                1 AS dict_match

            FROM ALM_TEST.WORK.promo_new_money_rate_dict d
                WITH (NOLOCK)

            WHERE s.is_mp = 0

              AND s.dt_open
                  BETWEEN d.date_from AND d.date_to

              AND s.termdays
                  BETWEEN d.term_min AND d.term_max

              AND d.conv_type =
                  CASE
                      WHEN ISNULL(s.conv,'AT_THE_END') = 'AT_THE_END'
                          THEN 'AT_THE_END'
                      ELSE 'NOT_AT_THE_END'
                  END

              AND s.OUT_RUB
                  BETWEEN d.amount_from AND d.amount_to

              AND ABS(s.rate_con - d.promo_rate) <= 0.0005
        ) d
    ),

    matched AS
    (
        SELECT
              j.*

            , CASE
                WHEN ISNULL(TRY_CAST(j.[Нов] AS int),0) = 1
                    THEN 1
                ELSE 0
              END AS is_nov

            , CASE
                WHEN ISNULL(TRY_CAST(j.[НДП] AS int),0) = 1
                  OR ISNULL(TRY_CAST(j.[НДМ] AS int),0) = 1
                    THEN 1
                ELSE 0
              END AS is_ndp_ndm

            , CASE
                WHEN ISNULL(TRY_CAST(j.[Пнс] AS int),0) = 1
                  OR ISNULL(TRY_CAST(j.[Мпл] AS int),0) = 1
                    THEN 1
                ELSE 0
              END AS is_pns_mpl

            , CASE
                WHEN ISNULL(TRY_CAST(j.[От1] AS int),0) = 1
                  OR ISNULL(TRY_CAST(j.[Пк2] AS int),0) = 1
                    THEN 1
                ELSE 0
              END AS retention

        FROM joined j
    ),

    classified AS
    (
        SELECT
              m.CON_ID
            , m.OUT_RUB
            , m.rate_con
            , m.termdays

            , CASE

                WHEN m.is_mp = 1
                    THEN N'Маркетплейсы'

                WHEN m.PROD_NAME_res = N'Активный процент'
                    THEN N'Активный процент'

                WHEN m.retention = 1
                    THEN N'Автоматический механизм удержания'

                WHEN m.is_nov = 1
                 AND m.dict_match = 1
                 AND m.is_pns_mpl = 0
                    THEN N'Новые клиенты'

                WHEN m.is_ndp_ndm = 1
                 AND m.dict_match = 1
                 AND m.is_pns_mpl = 0
                    THEN N'Новые деньги'

                WHEN m.special_2m = 1
                 AND m.is_pns_mpl = 0
                    THEN N'Ставка 15%'

                WHEN (m.is_nov = 1 OR m.is_ndp_ndm = 1)
                 AND m.dict_match IS NULL
                 AND m.is_pns_mpl = 0
                    THEN N'Новые деньги без справочника'

                WHEN m.dict_match = 1
                 AND m.is_pns_mpl = 0
                    THEN N'Другие вклады со ставкой новых денег'

                WHEN m.is_pns_mpl = 1
                    THEN N'Вклады ПНС и МПЛ'

                ELSE N'Остальные вклады'

              END AS segment_name

            , CASE
                WHEN m.termdays BETWEEN 26 AND 44 THEN 31
                WHEN m.termdays BETWEEN 45 AND 70 THEN 61
                WHEN m.termdays BETWEEN 85 AND 119 THEN 91
                WHEN m.termdays BETWEEN 120 AND 140 THEN 124
                WHEN m.termdays BETWEEN 141 AND 170 THEN 151
                WHEN m.termdays BETWEEN 175 AND 200 THEN 181
                WHEN m.termdays BETWEEN 201 AND 230 THEN 212
                WHEN m.termdays BETWEEN 231 AND 250 THEN 243
                WHEN m.termdays BETWEEN 251 AND 290 THEN 274
                WHEN m.termdays BETWEEN 340 AND 405 THEN 365
                WHEN m.termdays BETWEEN 540 AND 621 THEN 550
                WHEN m.termdays BETWEEN 720 AND 763 THEN 750
                WHEN m.termdays BETWEEN 1090 AND 1140 THEN 1100
                WHEN m.termdays BETWEEN 1450 AND 1475 THEN 1460
                WHEN m.termdays BETWEEN 1795 AND 1830 THEN 1825
                ELSE m.termdays
              END AS bucket_code

        FROM matched m
    ),

    agg AS
    (
        SELECT
              bucket_code

            , CASE
                WHEN GROUPING(segment_name) = 1
                    THEN N'Итого'
                ELSE segment_name
              END AS segment_name

            , GROUPING(segment_name) AS is_total

            , SUM(CAST(OUT_RUB AS decimal(24,2)))
                AS segment_vol

            , SUM(
                CAST(OUT_RUB AS decimal(24,2))
                * CAST(rate_con AS decimal(12,8))
              ) AS weighted_rate_sum

        FROM classified

        GROUP BY GROUPING SETS
        (
              (bucket_code, segment_name)
            , (bucket_code)
        )
    ),

    totals AS
    (
        SELECT
            SUM(segment_vol) AS grand_total
        FROM agg
        WHERE is_total = 0
    )

    SELECT
          a.bucket_code AS Bucket
        , a.segment_name AS Segment
        , a.segment_vol AS SegmentVolume

        , CAST(
            a.segment_vol * 1.0
            / NULLIF(t.grand_total, 0)
            AS decimal(18,4)
          ) AS SegmentSharePct

        , CAST(
            a.weighted_rate_sum
            / NULLIF(a.segment_vol, 0)
            AS decimal(9,6)
          ) AS ClientRate

    FROM agg a
    CROSS JOIN totals t
);
GO


SELECT *
FROM ALM_TEST.reports.fn_NewAttractionVolumes_forDimaMASS(
    '2026-11-02',
    '2026-10-01',
    '2026-11-02',
    1
)
ORDER BY Bucket, Segment;
