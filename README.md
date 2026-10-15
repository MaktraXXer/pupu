USE ALM_TEST;
GO

SET ANSI_NULLS ON;
GO

SET QUOTED_IDENTIFIER ON;
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
    /* ========================================================
       Список вкладов маркетплейсов
       ======================================================== */

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

    /* ========================================================
       Баланс

       Сразу отбираем только нужные открытия.
       Не выгружаем лишние поля.
       ======================================================== */

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
                      N'Достояние'
                  )
                    THEN 1
                ELSE 0
              END AS is_private

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

    /* ========================================================
       Надбавки вкладов для масс-сегмента

       Помню, что Всеволод говорил, что там бывают
       несколько записей по CON_ID.

       Поэтому берём только последнюю запись.

       Оптимизация:
       вместо ROW_NUMBER по всем договорам
       используем TOP (1) по конкретному CON_ID.

       Для маркетплейсов и private надбавки не нужны.
       ======================================================== */

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

                /* Атрибуты нужны только для масс-сегмента */
                AND s.is_mp = 0
                AND s.is_private = 0

            ORDER BY
                  a.DT_UPDATE DESC
                , a.loaddate DESC
        ) a

        /* ====================================================
           Джоиню справочник истории промо-ставки на новые деньги

           Нужен только факт совпадения.
           TOP (1) не даст размножить объём договора.

           is_active не проверяем:
           важна историческая дата открытия.
           ==================================================== */

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

    /* ========================================================
       Флаги надбавок

       NOV / NDP / NDM / MPL / PNS - новые деньги
       NOV / NDP / NDM - строгая проверка
       OT1 / PK2 - удержание
       ======================================================== */

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

    /* ========================================================
       Разделённые на категории вклады

       Приоритет:
       маркетплейс
       -> вклады УЧК
       -> масс как новые деньги
       -> масс на удержание
       -> масс "Мой дом безопций"
       -> остальные масс
       ======================================================== */

    classified AS
    (
        SELECT
              m.CON_ID
            , m.OUT_RUB
            , m.termdays

            /* Бизнес-категория */
            , CASE
                WHEN m.is_mp = 1
                    THEN N'вклады маркетплейса'

                WHEN m.is_private = 1
                    THEN N'вклад private-сегмента'

                WHEN m.dict_match = 1
                 AND m.new_money = 1
                    THEN N'вклад масс-сегмента с надбавкой равной новым деньгам'

                WHEN m.retention = 1
                    THEN N'вклад масс-сегмента на удержание'

                WHEN m.PROD_NAME_res = N'Мой дом безопций'
                    THEN N'вклад масс-сегмента базовый'

                ELSE N'нетипичные вклады масс-сегмента'
              END AS segment_name

            /* ====================================================
               Вызовется ошибка, если есть логическое нарушение.

               ERR 1:
               NOV/NDP/NDM, но нет справочника.

               MPL/PNS без справочника разрешены.

               ERR 2:
               Есть справочник, но отсутствуют все
               пять надбавок новых денег.

               ERR 3:
               Есть справочник + новые деньги + удержание.
               ==================================================== */

            , CASE

                /* ERR 1 */
                WHEN m.is_mp = 0
                 AND m.is_private = 0
                 AND m.strict_new_money = 1
                 AND m.dict_match IS NULL
                    THEN CONCAT(
                        'ERR_NOV_NDP_NDM_NO_DICT_CON_ID=',
                        m.CON_ID
                    )

                /* ERR 2 */
                WHEN m.is_mp = 0
                 AND m.is_private = 0
                 AND m.dict_match = 1
                 AND m.new_money = 0
                    THEN CONCAT(
                        'ERR_DICT_NO_NEW_MONEY_CON_ID=',
                        m.CON_ID
                    )

                /* ERR 3 */
                WHEN m.is_mp = 0
                 AND m.is_private = 0
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
                WHEN m.termdays BETWEEN 28 AND 44 THEN 31
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

    /* ========================================================
       Агрегация

       GROUPING SETS сразу считает:
       1. Объём по бакету и категории
       2. Итоговый объём бакета

       Ошибку проверяем один раз на агрегированную группу,
       а не для каждого договора в SUM().
       ======================================================== */

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

    /* ========================================================
       Доли

       segment_total - весь объём категории
       bucket_total  - весь объём бакета
       grand_total   - общий объём всех вкладов

       Всё считаем поверх уже агрегированных строк.
       ======================================================== */

    totals AS
    (
        SELECT
              bucket_code
            , segment_name
            , is_total
            , segment_vol

            , SUM(
                CASE
                    WHEN is_total = 0
                        THEN segment_vol
                    ELSE 0
                END
              ) OVER
              (
                  PARTITION BY segment_name
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

    /* ========================================================
       Резалт

       Формат сохраняем прежний для Excel / Power Query.
       ======================================================== */

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
