USE [ALM_TEST]
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

ALTER FUNCTION [reports].[fn_NewAttractionVolumes_TEST]
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
    /*
       Список вкладов маркетплейсов
    */

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

    /*
       Баланс
    */

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
                WHEN mp.prod_name_res IS NOT NULL
                    THEN 1
                ELSE 0
              END AS is_mp

            , CASE
                WHEN v.TSegmentName = N'ДЧБО'
                  OR v.PROD_NAME_res IN
                  (
                      N'Классический',
                      N'Привилегия',
                      N'Достояние',
                      N'Достояние без опций',
                      N'Достояние с пополнением и снятием',
                      N'Достояние (перем.ст.) без опций',
                      N'Достояние (перем.ст.) с пополнением и снятием'
                  )
                    THEN 1
                ELSE 0
              END AS is_private


            /* Особая ставка 15% на 2 месяца */
            , CASE
                WHEN v.termdays BETWEEN 45 AND 70
                 AND ISNULL(v.conv,'AT_THE_END') = 'AT_THE_END'
                 AND ABS(v.rate_con - 0.1500) <= 0.0005
                    THEN 1
                ELSE 0
              END AS special_2m


            /* Особая ставка 14.6% на 4 месяца */
            , CASE
                WHEN v.termdays BETWEEN 120 AND 140
                 AND ISNULL(v.conv,'AT_THE_END') = 'AT_THE_END'
                 AND ABS(v.rate_con - 0.1460) <= 0.0005
                    THEN 1
                ELSE 0
              END AS special_4m


        FROM ALM.ALM.vw_balance_rest_all v WITH (NOLOCK)

        LEFT JOIN mp_list mp
            ON mp.prod_name_res = v.PROD_NAME_res

        WHERE
            v.DT_REP = @ReportDate

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

            /* Исключаем договоры с плавающей ставкой */
            AND NOT EXISTS
            (
                SELECT 1
                FROM LIQUIDITY.liq.man_FloatContracts fc
                    WITH (NOLOCK)
                WHERE fc.CON_ID = v.CON_ID
            )

            AND
            (
                @ExcludeMP = 0
                OR mp.prod_name_res IS NULL
            )
    ),

    /*
       Надбавки вкладов для масс-сегмента

       Помню Всеволод говорил, что там бывают
       несколько записей по CON_ID.

       Поэтому берём только последнюю запись.

       Для маркетплейсов и private надбавки не нужны.
    */

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

            FROM ALM.ehd.attr_DepoFLConditions a
                WITH (NOLOCK)

            WHERE
                a.CON_ID = s.CON_ID
                AND s.is_mp = 0
                AND s.is_private = 0

            ORDER BY
                  a.DT_UPDATE DESC
                , a.loaddate DESC
        ) a


        /*
           Джоиню справочник истории промо-ставки
           на новые деньги.
        */

        OUTER APPLY
        (
            SELECT TOP (1)
                1 AS dict_match

            FROM ALM_TEST.WORK.promo_new_money_rate_dict d
                WITH (NOLOCK)

            WHERE
                s.is_mp = 0
                AND s.is_private = 0

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

    /*
       Флаги надбавок

       NOV / NDP / NDM / MPL / PNS - новые деньги
       NOV / NDP / NDM - строгая проверка
       OT1 / PK2 - удержание

       Если хотя бы одна надбавка равна 1,
       соответствующий флаг принимает значение 1.
    */

    matched AS
    (
        SELECT
              j.*

            , CASE
                WHEN
                       ISNULL(TRY_CAST(j.[Нов] AS int),0) = 1
                    OR ISNULL(TRY_CAST(j.[НДП] AS int),0) = 1
                    OR ISNULL(TRY_CAST(j.[НДМ] AS int),0) = 1
                    OR ISNULL(TRY_CAST(j.[Мпл] AS int),0) = 1
                    OR ISNULL(TRY_CAST(j.[Пнс] AS int),0) = 1
                THEN 1
                ELSE 0
              END AS new_money

            , CASE
                WHEN
                       ISNULL(TRY_CAST(j.[Нов] AS int),0) = 1
                    OR ISNULL(TRY_CAST(j.[НДП] AS int),0) = 1
                    OR ISNULL(TRY_CAST(j.[НДМ] AS int),0) = 1
                THEN 1
                ELSE 0
              END AS strict_new_money

            , CASE
                WHEN
                       ISNULL(TRY_CAST(j.[От1] AS int),0) = 1
                    OR ISNULL(TRY_CAST(j.[Пк2] AS int),0) = 1
                THEN 1
                ELSE 0
              END AS retention

        FROM joined j
    ),

    /*
       Разделённые на категории вклады

       Приоритет:

       маркетплейс
       -> особые предложения private
       -> остальные вклады private
       -> особые предложения масс
       -> Активный процент
       -> масс как новые деньги
       -> масс на удержание (От1 / Пк2)
       -> масс "Мой дом без опций"
       -> остальные масс
    */

    classified AS
    (
        SELECT
              m.CON_ID
            , m.OUT_RUB
            , m.termdays

            /* Бизнес-категория */
            , CASE

                WHEN m.is_mp = 1
                    THEN N'Маркетплейсы'


                /* Особые предложения private:
                   15% на 2 месяца
                   или 14.6% на 4 месяца */
                WHEN m.is_private = 1
                 AND
                 (
                     m.special_2m = 1
                     OR m.special_4m = 1
                 )
                    THEN N'Особые предложения private'


                /* Остальные private */
                WHEN m.is_private = 1
                    THEN N'private-сегмент'


                /* Особые предложения масс:
                   15% на 2 месяца */
                WHEN m.special_2m = 1
                    THEN N'Особые предложения масс'


                /* Активный процент */
                WHEN m.PROD_NAME_res = N'Активный процент'
                    THEN N'Активный процент'


                /* Новые деньги:
                   справочник + Нов / НДП / НДМ / Мпл / Пнс */
                WHEN m.dict_match = 1
                 AND m.new_money = 1
                    THEN N'Новые деньги'


                /* Механизм удержания:
                   достаточно От1 = 1 или Пк2 = 1 */
                WHEN m.retention = 1
                    THEN N'Механизм удержания'


                /* Базовая ставка */
                WHEN m.PROD_NAME_res IN
                (
                    N'Мой дом безопций',
                    N'Мой дом без опций'
                )
                    THEN N'Базовая ставка'


                ELSE N'Другие вклады'

              END AS segment_name


            /*
               Вызовется ошибка, если есть логическое нарушение
               - для теста.

               ERR 1:
               NOV/NDP/NDM, но нет справочника.

               MPL/PNS без справочника разрешены.

               ERR 2:
               Пока отключили проверку наличия справочника
               без надбавки новых денег.

               ERR 3:
               Есть справочник + новые деньги + удержание.

               ERR 4 (НОВОЕ):
               Одновременно присутствуют От1 / Пк2
               и Нов / НДП / НДМ.

               Для ERR 4 совпадение со справочником
               не требуется.

               Особые предложения масс исключены из
               прежних ERR 1 и ERR 3.

               Новая ERR 4 проверяет конфликт надбавок
               независимо от особой ставки.
            */

            , CASE

                /* ERR 4
                   Конфликт надбавок удержания
                   и Нов / НДП / НДМ */

                WHEN m.is_mp = 0
                 AND m.is_private = 0
                 AND m.retention = 1
                 AND m.strict_new_money = 1
                    THEN CONCAT(
                        'ERR_RETENTION_AND_NEW_MONEY_CON_ID=',
                        m.CON_ID
                    )


                /* ERR 1
                   Нов / НДП / НДМ, но нет справочника */

                WHEN m.is_mp = 0
                 AND m.is_private = 0
                 AND m.special_2m = 0
                 AND m.strict_new_money = 1
                 AND m.dict_match IS NULL
                    THEN CONCAT(
                        'ERR_NOV_NDP_NDM_NO_DICT_CON_ID=',
                        m.CON_ID
                    )


                /*
                -- ERR 2 временно отключён

                WHEN m.is_mp = 0
                 AND m.is_private = 0
                 AND m.dict_match = 1
                 AND m.new_money = 0
                    THEN CONCAT(
                        'ERR_DICT_NO_NEW_MONEY_CON_ID=',
                        m.CON_ID
                    )
                */


                /* ERR 3
                   Справочник + новые деньги + удержание */

                WHEN m.is_mp = 0
                 AND m.is_private = 0
                 AND m.special_2m = 0
                 AND m.dict_match = 1
                 AND m.new_money = 1
                 AND m.retention = 1
                    THEN CONCAT(
                        'ERR_NEW_MONEY_AND_RETENTION_CON_ID=',
                        m.CON_ID
                    )

                ELSE '0'
              END AS validation_code


            /* Бакеты срочности для селекта */
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

    /*
       Агрегация

       Объём каждой категории по бакету.
       Отдельно общий объём каждого бакета.

       Если есть логическая ошибка, прерываем расчёт.
    */

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

            , SUM(OUT_RUB)
              + CONVERT(int, MAX(validation_code))
              AS segment_vol

        FROM classified

        GROUP BY GROUPING SETS
        (
              (bucket_code, segment_name)
            , (bucket_code)
        )
    ),

    /*
       Доли

       Private:
       Особые предложения private + private-сегмент
       считаем от общего объёма private.

       Остальные категории:
       считаем от общего объёма остальных вкладов.

       Маркетплейсы входят в общий знаменатель
       остальных категорий, если @ExcludeMP = 0.

       BucketSharePct:
       от всего объёма привлечений.
    */

    totals AS
    (
        SELECT
              bucket_code
            , segment_name
            , is_total
            , segment_vol

            /* Общий знаменатель:
               отдельно весь private,
               отдельно все остальные категории */
            , SUM(
                CASE
                    WHEN is_total = 0
                        THEN segment_vol
                    ELSE 0
                END
              ) OVER
              (
                  PARTITION BY
                      CASE
                          WHEN segment_name IN
                          (
                              N'private-сегмент',
                              N'Особые предложения private'
                          )
                              THEN 1
                          ELSE 0
                      END
              ) AS segment_total

            , MAX(
                CASE
                    WHEN is_total = 1
                        THEN segment_vol
                END
              ) OVER
              (
                  PARTITION BY bucket_code
              ) AS bucket_total

            , SUM(
                CASE
                    WHEN is_total = 1
                        THEN segment_vol
                    ELSE 0
                END
              ) OVER () AS grand_total

        FROM agg
    )

    /*
       Резалт
    */

    SELECT
          bucket_code AS Bucket
        , segment_name AS Segment
        , segment_vol AS SegmentVolume

        , CAST(
            CASE
                WHEN is_total = 1
                    THEN segment_vol / NULLIF(grand_total,0)
                ELSE segment_vol / NULLIF(segment_total,0)
            END
            AS decimal(18,4)
          ) AS SegmentSharePct

        , bucket_total AS BucketVolume

        , CAST(
            bucket_total / NULLIF(grand_total,0)
            AS decimal(18,4)
          ) AS BucketSharePct

    FROM totals
);
GO



DECLARE @dt_to date = DATEADD(day,-2,CAST(GETDATE() AS date));
DECLARE @dt_from date = DATEADD(day,-8,CAST(GETDATE() AS date));

SELECT *
FROM ALM_TEST.reports.fn_NewAttractionVolumes_TEST
(
    @dt_to,
    @dt_from,
    @dt_to,
    0
)
OPTION (RECOMPILE);
