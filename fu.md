USE [ALM];
SET NOCOUNT ON;


/* ============================================================
   ПАРАМЕТРЫ

   Скрипт последовательно обработает КАЖДУЮ календарную дату
   от @DateFrom до @DateTo включительно.

   Если на какую-либо дату баланса нет, дата просто пропускается.
   ============================================================ */

DECLARE @DateFrom date = '2025-09-30';
DECLARE @DateTo   date = '2026-08-24';

DECLARE @CurrentDate date = @DateFrom;


/* ============================================================
   ВРЕМЕННЫЕ ТАБЛИЦЫ

   Они создаются ОДИН РАЗ.
   В цикле используются через TRUNCATE.

   Поэтому tempdb не копит историю балансов.
   ============================================================ */

IF OBJECT_ID('tempdb..#bal_day') IS NOT NULL
    DROP TABLE #bal_day;

IF OBJECT_ID('tempdb..#con_ids') IS NOT NULL
    DROP TABLE #con_ids;

IF OBJECT_ID('tempdb..#attr_flags') IS NOT NULL
    DROP TABLE #attr_flags;

IF OBJECT_ID('tempdb..#result') IS NOT NULL
    DROP TABLE #result;


/* ============================================================
   Баланс только ОДНОЙ даты
   ============================================================ */

CREATE TABLE #bal_day
(
      con_id bigint NULL
    , PROD_NAME_res nvarchar(500) NULL
    , out_rub decimal(38,6) NOT NULL
    , is_floatrate bit NOT NULL
);


/* Индекс пригодится при соединении с признаками договора */
CREATE NONCLUSTERED INDEX IX_bal_day_con_id
ON #bal_day (con_id);


/* ============================================================
   Только con_id текущего дня
   ============================================================ */

CREATE TABLE #con_ids
(
    con_id bigint NOT NULL
        PRIMARY KEY CLUSTERED
);


/* ============================================================
   Признаки только договоров текущего дня
   ============================================================ */

CREATE TABLE #attr_flags
(
      con_id bigint NOT NULL
        PRIMARY KEY CLUSTERED

    , is_nov_flag bit NOT NULL
    , is_pr2_pr3_flag bit NOT NULL
    , is_ndp_ndm_flag bit NOT NULL
    , is_pk2_ot1_flag bit NOT NULL
    , is_pk3_pk6_flag bit NOT NULL
    , is_mpl_flag bit NOT NULL
    , is_pns_flag bit NOT NULL
);


/* ============================================================
   Маленькая итоговая таблица

   Одна строка = одна дата.
   ============================================================ */

CREATE TABLE #result
(
      dt_rep date NOT NULL
        PRIMARY KEY CLUSTERED


    /* 9 специальных категорий */

    , balance_float decimal(38,6) NOT NULL

    , balance_fu decimal(38,6) NOT NULL

    , balance_nov decimal(38,6) NOT NULL

    , balance_pr2_pr3 decimal(38,6) NOT NULL

    , balance_ndp_ndm decimal(38,6) NOT NULL

    , balance_pk2_ot1 decimal(38,6) NOT NULL

    , balance_pk3_pk6 decimal(38,6) NOT NULL

    , balance_mpl decimal(38,6) NOT NULL

    , balance_pns decimal(38,6) NOT NULL


    /* Остальные */

    , balance_other decimal(38,6) NOT NULL


    /* Весь срочный баланс */

    , balance_total decimal(38,6) NOT NULL
);



/* ============================================================
   ЦИКЛ ПО ДАТАМ
   ============================================================ */

WHILE @CurrentDate <= @DateTo
BEGIN


    /* ========================================================
       1. Очищаем рабочие таблицы прошлого дня

       Физически таблицы не пересоздаются.
       ======================================================== */

    TRUNCATE TABLE #bal_day;
    TRUNCATE TABLE #con_ids;
    TRUNCATE TABLE #attr_flags;



    /* ========================================================
       2. БАЛАНС ТОЛЬКО НА ОДНУ ДАТУ

       Это принципиальный момент:
           dt_rep = @CurrentDate

       Никаких BETWEEN по VW_balance_rest_all.
       ======================================================== */

    INSERT INTO #bal_day
    (
          con_id
        , PROD_NAME_res
        , out_rub
        , is_floatrate
    )

    SELECT
          TRY_CAST(t.con_id AS bigint)

        , t.PROD_NAME_res

        , CAST(t.out_rub AS decimal(38,6))

        , CASE
              WHEN ISNULL(
                       TRY_CAST(t.is_floatrate AS int),
                       0
                   ) = 1
                  THEN 1
              ELSE 0
          END

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    WHERE
        t.dt_rep = @CurrentDate

        AND t.section_name = N'Срочные'

        AND t.block_name = N'Привлечение ФЛ'

        AND t.acc_role = N'LIAB'

        AND t.od_flag = 1

        AND t.cur = '810'

        AND t.out_rub IS NOT NULL

        AND t.out_rub >= 0

    OPTION (RECOMPILE);



    /* ========================================================
       Если на эту дату баланса вообще нет —
       просто переходим к следующему дню.

       Нулевую строку намеренно НЕ создаём:
       отсутствие данных не должно выглядеть как нулевой баланс.
       ======================================================== */

    IF EXISTS
    (
        SELECT 1
        FROM #bal_day
    )
    BEGIN


        /* ====================================================
           3. con_id только текущего snapshot
           ==================================================== */

        INSERT INTO #con_ids
        (
            con_id
        )

        SELECT DISTINCT
            con_id

        FROM #bal_day

        WHERE
            con_id IS NOT NULL;



        /* ====================================================
           4. Последняя запись attr_DepoFLConditions

           Берём признаки ТОЛЬКО для договоров,
           присутствующих сегодня.

           По каждому con_id:
               DT_UPDATE DESC
               loaddate DESC
           ==================================================== */

        ;WITH attr_ranked AS
        (
            SELECT
                  TRY_CAST(a.CON_ID AS bigint) AS con_id


                /* Нов */
                , CASE
                      WHEN ISNULL(
                               TRY_CAST(a.[Нов] AS int),
                               0
                           ) = 1
                          THEN 1
                      ELSE 0
                  END AS is_nov_flag


                /* Пр2 / Пр3 */
                , CASE
                      WHEN ISNULL(
                               TRY_CAST(a.[Пр2] AS int),
                               0
                           ) = 1

                        OR ISNULL(
                               TRY_CAST(a.[Пр3] AS int),
                               0
                           ) = 1

                          THEN 1
                      ELSE 0
                  END AS is_pr2_pr3_flag


                /* НДП / НДМ */
                , CASE
                      WHEN ISNULL(
                               TRY_CAST(a.[НДП] AS int),
                               0
                           ) = 1

                        OR ISNULL(
                               TRY_CAST(a.[НДМ] AS int),
                               0
                           ) = 1

                          THEN 1
                      ELSE 0
                  END AS is_ndp_ndm_flag


                /* Пк2 / От1 */
                , CASE
                      WHEN ISNULL(
                               TRY_CAST(a.[Пк2] AS int),
                               0
                           ) = 1

                        OR ISNULL(
                               TRY_CAST(a.[От1] AS int),
                               0
                           ) = 1

                          THEN 1
                      ELSE 0
                  END AS is_pk2_ot1_flag


                /* Пк3 / Пк6 */
                , CASE
                      WHEN ISNULL(
                               TRY_CAST(a.[Пк3] AS int),
                               0
                           ) = 1

                        OR ISNULL(
                               TRY_CAST(a.[Пк6] AS int),
                               0
                           ) = 1

                          THEN 1
                      ELSE 0
                  END AS is_pk3_pk6_flag


                /* МПЛ */
                , CASE
                      WHEN ISNULL(
                               TRY_CAST(a.[Мпл] AS int),
                               0
                           ) = 1
                          THEN 1
                      ELSE 0
                  END AS is_mpl_flag


                /* Пнс */
                , CASE
                      WHEN ISNULL(
                               TRY_CAST(a.[Пнс] AS int),
                               0
                           ) = 1
                          THEN 1
                      ELSE 0
                  END AS is_pns_flag


                , ROW_NUMBER() OVER
                  (
                      PARTITION BY a.CON_ID

                      ORDER BY
                            a.DT_UPDATE DESC
                          , a.loaddate DESC
                  ) AS rn


            FROM alm.ehd.attr_DepoFLConditions a
                 WITH (NOLOCK)

            INNER JOIN #con_ids c
                ON c.con_id =
                   TRY_CAST(a.CON_ID AS bigint)
        )


        INSERT INTO #attr_flags
        (
              con_id

            , is_nov_flag
            , is_pr2_pr3_flag
            , is_ndp_ndm_flag
            , is_pk2_ot1_flag
            , is_pk3_pk6_flag
            , is_mpl_flag
            , is_pns_flag
        )

        SELECT
              con_id

            , is_nov_flag
            , is_pr2_pr3_flag
            , is_ndp_ndm_flag
            , is_pk2_ot1_flag
            , is_pk3_pk6_flag
            , is_mpl_flag
            , is_pns_flag

        FROM attr_ranked

        WHERE rn = 1;



        /* ====================================================
           5. КЛАССИФИКАЦИЯ БАЛАНСА

           ЖЁСТКИЙ ПРИОРИТЕТ:

           1. FLOAT
           2. ФУ
           3. Нов
           4. Пр2 / Пр3
           5. НДП / НДМ
           6. Пк2 / От1
           7. Пк3 / Пк6
           8. МПЛ
           9. Пнс
           10. Остальные

           Каждый рубль попадает только в ОДНУ категорию.
           ==================================================== */

        ;WITH classified AS
        (
            SELECT
                  b.out_rub

                , CASE

                      /* ========================================
                         1. ПЛАВАЮЩИЙ ВКЛАД

                         Абсолютно первый приоритет.
                         ======================================== */

                      WHEN b.is_floatrate = 1
                          THEN N'float'


                      /* ========================================
                         2. ФИНУСЛУГИ
                         ======================================== */

                      WHEN b.PROD_NAME_res IN
                      (
                            N'Надёжный прайм'
                          , N'Надёжный VIP'
                          , N'Надёжный премиум'
                          , N'Надёжный промо'
                          , N'Надёжный старт'
                          , N'Надёжный Т2'
                          , N'Надёжный Мегафон'
                          , N'Надёжный процент'
                          , N'Надёжныйпроцент'
                          , N'Могучий'
                          , N'Надёжный'
                      )
                          THEN N'fu'


                      /* ========================================
                         3. НОВ
                         ======================================== */

                      WHEN ISNULL(
                               a.is_nov_flag,
                               0
                           ) = 1
                          THEN N'nov'


                      /* ========================================
                         4. Пр2 / Пр3
                         ======================================== */

                      WHEN ISNULL(
                               a.is_pr2_pr3_flag,
                               0
                           ) = 1
                          THEN N'pr2_pr3'


                      /* ========================================
                         5. НДП / НДМ
                         ======================================== */

                      WHEN ISNULL(
                               a.is_ndp_ndm_flag,
                               0
                           ) = 1
                          THEN N'ndp_ndm'


                      /* ========================================
                         6. Пк2 / От1
                         ======================================== */

                      WHEN ISNULL(
                               a.is_pk2_ot1_flag,
                               0
                           ) = 1
                          THEN N'pk2_ot1'


                      /* ========================================
                         7. Пк3 / Пк6
                         ======================================== */

                      WHEN ISNULL(
                               a.is_pk3_pk6_flag,
                               0
                           ) = 1
                          THEN N'pk3_pk6'


                      /* ========================================
                         8. МПЛ
                         ======================================== */

                      WHEN ISNULL(
                               a.is_mpl_flag,
                               0
                           ) = 1
                          THEN N'mpl'


                      /* ========================================
                         9. Пнс
                         ======================================== */

                      WHEN ISNULL(
                               a.is_pns_flag,
                               0
                           ) = 1
                          THEN N'pns'


                      /* ========================================
                         10. Остальные
                         ======================================== */

                      ELSE N'other'

                  END AS deposit_category


            FROM #bal_day b

            LEFT JOIN #attr_flags a
                ON a.con_id = b.con_id
        )


        /* ====================================================
           6. Сразу превращаем большой дневной баланс
              в ОДНУ строку.

           После этого дневные данные нам больше не нужны.
           ==================================================== */

        INSERT INTO #result
        (
              dt_rep

            , balance_float
            , balance_fu
            , balance_nov
            , balance_pr2_pr3
            , balance_ndp_ndm
            , balance_pk2_ot1
            , balance_pk3_pk6
            , balance_mpl
            , balance_pns

            , balance_other

            , balance_total
        )

        SELECT
              @CurrentDate


            /* 1. FLOAT */
            , SUM(
                  CASE
                      WHEN deposit_category = N'float'
                          THEN out_rub
                      ELSE 0
                  END
              )


            /* 2. ФУ */
            , SUM(
                  CASE
                      WHEN deposit_category = N'fu'
                          THEN out_rub
                      ELSE 0
                  END
              )


            /* 3. Нов */
            , SUM(
                  CASE
                      WHEN deposit_category = N'nov'
                          THEN out_rub
                      ELSE 0
                  END
              )


            /* 4. Пр2 / Пр3 */
            , SUM(
                  CASE
                      WHEN deposit_category = N'pr2_pr3'
                          THEN out_rub
                      ELSE 0
                  END
              )


            /* 5. НДП / НДМ */
            , SUM(
                  CASE
                      WHEN deposit_category = N'ndp_ndm'
                          THEN out_rub
                      ELSE 0
                  END
              )


            /* 6. Пк2 / От1 */
            , SUM(
                  CASE
                      WHEN deposit_category = N'pk2_ot1'
                          THEN out_rub
                      ELSE 0
                  END
              )


            /* 7. Пк3 / Пк6 */
            , SUM(
                  CASE
                      WHEN deposit_category = N'pk3_pk6'
                          THEN out_rub
                      ELSE 0
                  END
              )


            /* 8. МПЛ */
            , SUM(
                  CASE
                      WHEN deposit_category = N'mpl'
                          THEN out_rub
                      ELSE 0
                  END
              )


            /* 9. Пнс */
            , SUM(
                  CASE
                      WHEN deposit_category = N'pns'
                          THEN out_rub
                      ELSE 0
                  END
              )


            /* Остальные */
            , SUM(
                  CASE
                      WHEN deposit_category = N'other'
                          THEN out_rub
                      ELSE 0
                  END
              )


            /* Весь баланс */
            , SUM(out_rub)


        FROM classified

        OPTION (RECOMPILE);

    END;



    /* ========================================================
       7. Переходим к следующей календарной дате
       ======================================================== */

    SET @CurrentDate =
        DATEADD(day, 1, @CurrentDate);

END;


/* ============================================================
   8. ИТОГ

   Небольшая таблица:
   одна строка на дату.
   ============================================================ */

SELECT
      dt_rep

    , balance_float
    , balance_fu
    , balance_nov
    , balance_pr2_pr3
    , balance_ndp_ndm
    , balance_pk2_ot1
    , balance_pk3_pk6
    , balance_mpl
    , balance_pns

    , balance_other

    , balance_total


    /* ========================================================
       Контроль.

       Должен ВСЕГДА быть 0.
       ======================================================== */

    , balance_total
      -
      (
            balance_float
          + balance_fu
          + balance_nov
          + balance_pr2_pr3
          + balance_ndp_ndm
          + balance_pk2_ot1
          + balance_pk3_pk6
          + balance_mpl
          + balance_pns
          + balance_other
      ) AS control_diff


FROM #result

ORDER BY
    dt_rep;
