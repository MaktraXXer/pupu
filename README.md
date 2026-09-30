Да. Здесь правильнее **не удалять таблицу**: текущий `PRIMARY KEY (observation_month, cli_id)` полезен и это единственный явно заданный индекс. Просто добавим 8 новых полей. Исторические строки получат в них `0`; если нужны корректные PK2/OT1 за старые месяцы, эти месяцы затем надо перезапустить с `@ReplaceExisting = 1`.

Клиентские категории после изменения будут взаимоисключающими с приоритетом:

```text
01. Вкладчики к выходу
02. НС без вкладов к выходу
03. Открывшие вклад
04. Открывшие НС
```

Для `03` и `04` факт открытия определяется той же архитектурой, что уже используется процедурой: продукт присутствует на `#bal_end`, а `dt_open` попадает внутрь текущего периода. Старые `01/02` вообще не меняются.

Новые PK2/OT1-поля — **только детализация** существующей категории `pk2_ot1`. Общие `exit_pk2_ot1_td_sum`, `opened_pk2_ot1` и их флаги сохраняются неизменными. Если одновременно `Пк2=1` и `От1=1`, для детализации договор относится в `Пк2`, поэтому двойного счёта нет:

```text
exit_pk2_ot1_td_sum = exit_pk2_td_sum + exit_ot1_td_sum
opened_pk2_ot1      = opened_pk2 + opened_ot1
```

## А. Обновление таблицы

```sql
USE [ALM_TEST];
GO


/* ============================================================
   Добавляем детализацию общей категории Пк2 / От1.

   Существующие данные НЕ удаляются.
   Старым строкам новые показатели устанавливаются в 0.
   ============================================================ */


/* ---------- Вклады к выходу: суммы ---------- */

IF COL_LENGTH(
       'alm_report.depo_fl_client_monthly_stats',
       'exit_pk2_td_sum'
   ) IS NULL
BEGIN
    ALTER TABLE [alm_report].[depo_fl_client_monthly_stats]
    ADD [exit_pk2_td_sum] decimal(38,6) NOT NULL
        CONSTRAINT [DF_depo_fl_stats_exit_pk2_td_sum]
        DEFAULT (0) WITH VALUES;
END;
GO


IF COL_LENGTH(
       'alm_report.depo_fl_client_monthly_stats',
       'exit_ot1_td_sum'
   ) IS NULL
BEGIN
    ALTER TABLE [alm_report].[depo_fl_client_monthly_stats]
    ADD [exit_ot1_td_sum] decimal(38,6) NOT NULL
        CONSTRAINT [DF_depo_fl_stats_exit_ot1_td_sum]
        DEFAULT (0) WITH VALUES;
END;
GO


/* ---------- Вклады к выходу: флаги ---------- */

IF COL_LENGTH(
       'alm_report.depo_fl_client_monthly_stats',
       'has_pk2_exit_td_flag'
   ) IS NULL
BEGIN
    ALTER TABLE [alm_report].[depo_fl_client_monthly_stats]
    ADD [has_pk2_exit_td_flag] bit NOT NULL
        CONSTRAINT [DF_depo_fl_stats_has_pk2_exit]
        DEFAULT (0) WITH VALUES;
END;
GO


IF COL_LENGTH(
       'alm_report.depo_fl_client_monthly_stats',
       'has_ot1_exit_td_flag'
   ) IS NULL
BEGIN
    ALTER TABLE [alm_report].[depo_fl_client_monthly_stats]
    ADD [has_ot1_exit_td_flag] bit NOT NULL
        CONSTRAINT [DF_depo_fl_stats_has_ot1_exit]
        DEFAULT (0) WITH VALUES;
END;
GO


/* ---------- Открытые вклады: суммы ---------- */

IF COL_LENGTH(
       'alm_report.depo_fl_client_monthly_stats',
       'opened_pk2'
   ) IS NULL
BEGIN
    ALTER TABLE [alm_report].[depo_fl_client_monthly_stats]
    ADD [opened_pk2] decimal(38,6) NOT NULL
        CONSTRAINT [DF_depo_fl_stats_opened_pk2]
        DEFAULT (0) WITH VALUES;
END;
GO


IF COL_LENGTH(
       'alm_report.depo_fl_client_monthly_stats',
       'opened_ot1'
   ) IS NULL
BEGIN
    ALTER TABLE [alm_report].[depo_fl_client_monthly_stats]
    ADD [opened_ot1] decimal(38,6) NOT NULL
        CONSTRAINT [DF_depo_fl_stats_opened_ot1]
        DEFAULT (0) WITH VALUES;
END;
GO


/* ---------- Открытые вклады: флаги ---------- */

IF COL_LENGTH(
       'alm_report.depo_fl_client_monthly_stats',
       'has_opened_pk2_flag'
   ) IS NULL
BEGIN
    ALTER TABLE [alm_report].[depo_fl_client_monthly_stats]
    ADD [has_opened_pk2_flag] bit NOT NULL
        CONSTRAINT [DF_depo_fl_stats_has_opened_pk2]
        DEFAULT (0) WITH VALUES;
END;
GO


IF COL_LENGTH(
       'alm_report.depo_fl_client_monthly_stats',
       'has_opened_ot1_flag'
   ) IS NULL
BEGIN
    ALTER TABLE [alm_report].[depo_fl_client_monthly_stats]
    ADD [has_opened_ot1_flag] bit NOT NULL
        CONSTRAINT [DF_depo_fl_stats_has_opened_ot1]
        DEFAULT (0) WITH VALUES;
END;
GO
```

## Б. Новая процедура целиком

```sql
USE [ALM_TEST];
GO

SET ANSI_NULLS ON;
GO

SET QUOTED_IDENTIFIER ON;
GO


ALTER PROCEDURE
    [alm_report].[usp_load_depo_fl_client_monthly_stats]

      @StartBaseDate    date
    , @FinalEndDate     date
    , @ReplaceExisting  bit = 1

AS
BEGIN

    SET NOCOUNT ON;
    SET XACT_ABORT ON;


    /* ========================================================
       0. Проверка параметров
       ======================================================== */

    IF @StartBaseDate IS NULL
       OR @FinalEndDate IS NULL
    BEGIN
        THROW 51000,
              N'Необходимо передать @StartBaseDate и @FinalEndDate.',
              1;
    END;


    IF @FinalEndDate <= @StartBaseDate
    BEGIN
        THROW 51001,
              N'@FinalEndDate должен быть больше @StartBaseDate.',
              1;
    END;


    IF @StartBaseDate <> EOMONTH(@StartBaseDate)
    BEGIN
        THROW 51002,
              N'@StartBaseDate должен быть последним календарным днём месяца.',
              1;
    END;



    /* ========================================================
       Исключённые клиенты
       ======================================================== */

    CREATE TABLE #excluded_clients
    (
        cli_id bigint NOT NULL PRIMARY KEY
    );


    INSERT INTO #excluded_clients (cli_id)
    VALUES
          (4198051)
        , (3935060394)
        , (3929042989)
        , (3934007920)
        , (3926799230)
        , (3927382973);



    /* ========================================================
       Переменные периода
       ======================================================== */

    DECLARE @BaseDate date = @StartBaseDate;
    DECLARE @EndDate date;

    DECLARE @ExitFrom date;
    DECLARE @ExitTo date;

    DECLARE @OpenFrom date;
    DECLARE @OpenTo date;

    DECLARE @ObservationMonth date;
    DECLARE @IsPartialMonth bit;

    DECLARE @ErrorMessage nvarchar(2048);



    /* ========================================================
       1. Первый snapshot
       ======================================================== */

    SELECT
          CAST(t.cli_id AS bigint) AS cli_id
        , CAST(t.con_id AS bigint) AS con_id
        , CAST(t.dt_open AS date) AS dt_open
        , CAST(t.dt_close_plan AS date) AS dt_close_plan
        , t.section_name
        , t.PROD_NAME_res
        , CAST(t.out_rub AS decimal(38,6)) AS out_rub
        , t.TSEGMENTNAME

    INTO #bal_base

    FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)

    WHERE
        t.dt_rep = @BaseDate

        AND t.section_name IN
        (
              N'Срочные'
            , N'Накопительный счёт'
        )

        AND t.block_name = N'Привлечение ФЛ'
        AND t.acc_role   = N'LIAB'
        AND t.od_flag    = 1
        AND t.cur        = '810'

        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0

    OPTION (RECOMPILE);


    IF NOT EXISTS
    (
        SELECT 1
        FROM #bal_base
    )
    BEGIN

        SET @ErrorMessage =
            N'Не найден баланс на начальную дату '
            + CONVERT(nvarchar(10), @BaseDate, 120);

        THROW 51003, @ErrorMessage, 1;

    END;



    /* Следующий snapshot */
    SELECT TOP (0)
        *
    INTO #bal_end
    FROM #bal_base;



    /* ========================================================
       2. СКОЛЬЗЯЩИЙ ЦИКЛ
       ======================================================== */

    WHILE @BaseDate < @FinalEndDate
    BEGIN


        /* ====================================================
           Конец следующего периода
           ==================================================== */

        SET @EndDate =
            EOMONTH(
                DATEADD(month, 1, @BaseDate)
            );


        IF @EndDate > @FinalEndDate
            SET @EndDate = @FinalEndDate;


        SET @ExitFrom = DATEADD(day, 1, @BaseDate);
        SET @ExitTo   = @EndDate;

        SET @OpenFrom = DATEADD(day, 1, @BaseDate);
        SET @OpenTo   = @EndDate;


        SET @ObservationMonth =
            DATEFROMPARTS
            (
                  YEAR(@EndDate)
                , MONTH(@EndDate)
                , 1
            );


        SET @IsPartialMonth =
            CASE
                WHEN @EndDate <> EOMONTH(@EndDate)
                    THEN 1
                ELSE 0
            END;



        /* ====================================================
           3. Следующий snapshot
           ==================================================== */

        TRUNCATE TABLE #bal_end;


        INSERT INTO #bal_end
        (
              cli_id
            , con_id
            , dt_open
            , dt_close_plan
            , section_name
            , PROD_NAME_res
            , out_rub
            , TSEGMENTNAME
        )

        SELECT
              CAST(t.cli_id AS bigint)
            , CAST(t.con_id AS bigint)
            , CAST(t.dt_open AS date)
            , CAST(t.dt_close_plan AS date)
            , t.section_name
            , t.PROD_NAME_res
            , CAST(t.out_rub AS decimal(38,6))
            , t.TSEGMENTNAME

        FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)

        WHERE
            t.dt_rep = @EndDate

            AND t.section_name IN
            (
                  N'Срочные'
                , N'Накопительный счёт'
            )

            AND t.block_name = N'Привлечение ФЛ'
            AND t.acc_role   = N'LIAB'
            AND t.od_flag    = 1
            AND t.cur        = '810'

            AND t.out_rub IS NOT NULL
            AND t.out_rub >= 0

        OPTION (RECOMPILE);



        IF NOT EXISTS
        (
            SELECT 1
            FROM #bal_end
        )
        BEGIN

            SET @ErrorMessage =
                N'Не найден баланс на конечную дату '
                + CONVERT(nvarchar(10), @EndDate, 120);

            THROW 51004, @ErrorMessage, 1;

        END;



        /* ====================================================
           Уже загруженный месяц
           ==================================================== */

        IF @ReplaceExisting = 0
           AND EXISTS
           (
               SELECT 1

               FROM
                   [alm_report].[depo_fl_client_monthly_stats]

               WHERE
                   observation_month = @ObservationMonth
           )
        BEGIN

            TRUNCATE TABLE #bal_base;


            INSERT INTO #bal_base
            SELECT *
            FROM #bal_end;


            SET @BaseDate = @EndDate;

            CONTINUE;

        END;



        /* ====================================================
           4. БАЗА КЛИЕНТОВ

           Приоритет категорий:

           01. вклад к выходу
           02. НС на начало без вклада к выходу
           03. открыл срочный вклад
           04. открыл НС

           Категории взаимоисключающие.
           ==================================================== */

        IF OBJECT_ID('tempdb..#client_scope') IS NOT NULL
            DROP TABLE #client_scope;


        ;WITH exit_clients AS
        (
            SELECT DISTINCT
                b.cli_id

            FROM #bal_base b

            WHERE
                b.section_name = N'Срочные'

                AND b.dt_close_plan >= @ExitFrom
                AND b.dt_close_plan <= @ExitTo
        ),


        ns_base_clients AS
        (
            SELECT DISTINCT
                b.cli_id

            FROM #bal_base b

            WHERE
                b.section_name = N'Накопительный счёт'
        ),


        opened_td_clients AS
        (
            SELECT DISTINCT
                b.cli_id

            FROM #bal_end b

            WHERE
                b.section_name = N'Срочные'

                AND b.dt_open >= @OpenFrom
                AND b.dt_open <= @OpenTo
        ),


        opened_ns_clients AS
        (
            SELECT DISTINCT
                b.cli_id

            FROM #bal_end b

            WHERE
                b.section_name = N'Накопительный счёт'

                AND b.dt_open >= @OpenFrom
                AND b.dt_open <= @OpenTo
        ),


        scope_union AS
        (
            /* ================================================
               01. Вкладчики к выходу
               ================================================ */

            SELECT
                  e.cli_id

                , CAST(
                      N'01. Вкладчики к выходу'
                      AS nvarchar(100)
                  ) AS client_base_type

            FROM exit_clients e


            UNION ALL


            /* ================================================
               02. НС без вкладов к выходу
               ================================================ */

            SELECT
                  n.cli_id

                , CAST(
                      N'02. НС без вкладов к выходу'
                      AS nvarchar(100)
                  )

            FROM ns_base_clients n

            WHERE NOT EXISTS
            (
                SELECT 1
                FROM exit_clients e
                WHERE e.cli_id = n.cli_id
            )


            UNION ALL


            /* ================================================
               03. Открывшие вклад

               Нет вклада к выходу.
               Нет НС на начало.
               В течение периода открыт срочный вклад.
               ================================================ */

            SELECT
                  o.cli_id

                , CAST(
                      N'03. Открывшие вклад'
                      AS nvarchar(100)
                  )

            FROM opened_td_clients o

            WHERE NOT EXISTS
            (
                SELECT 1
                FROM exit_clients e
                WHERE e.cli_id = o.cli_id
            )

            AND NOT EXISTS
            (
                SELECT 1
                FROM ns_base_clients n
                WHERE n.cli_id = o.cli_id
            )


            UNION ALL


            /* ================================================
               04. Открывшие НС

               Нет вклада к выходу.
               Нет НС на начало.
               Не попал в "Открывшие вклад".
               В периоде открыт НС.
               ================================================ */

            SELECT
                  n.cli_id

                , CAST(
                      N'04. Открывшие НС'
                      AS nvarchar(100)
                  )

            FROM opened_ns_clients n

            WHERE NOT EXISTS
            (
                SELECT 1
                FROM exit_clients e
                WHERE e.cli_id = n.cli_id
            )

            AND NOT EXISTS
            (
                SELECT 1
                FROM ns_base_clients nb
                WHERE nb.cli_id = n.cli_id
            )

            AND NOT EXISTS
            (
                SELECT 1
                FROM opened_td_clients td
                WHERE td.cli_id = n.cli_id
            )
        )


        SELECT
              x.cli_id
            , x.client_base_type

        INTO #client_scope

        FROM scope_union x

        WHERE NOT EXISTS
        (
            SELECT 1

            FROM #excluded_clients ex

            WHERE
                ex.cli_id = x.cli_id
        );


        CREATE UNIQUE CLUSTERED INDEX
            IX_client_scope_cli
        ON #client_scope (cli_id);



        /* ====================================================
           5. Только необходимые con_id
           ==================================================== */

        IF OBJECT_ID('tempdb..#relevant_con_ids') IS NOT NULL
            DROP TABLE #relevant_con_ids;


        SELECT
            r.con_id

        INTO #relevant_con_ids

        FROM
        (
            SELECT
                b.con_id

            FROM #bal_base b

            INNER JOIN #client_scope c
                ON c.cli_id = b.cli_id

            WHERE
                b.con_id IS NOT NULL


            UNION


            SELECT
                e.con_id

            FROM #bal_end e

            INNER JOIN #client_scope c
                ON c.cli_id = e.cli_id

            WHERE
                e.con_id IS NOT NULL

        ) r;


        CREATE UNIQUE CLUSTERED INDEX
            IX_relevant_con_ids
        ON #relevant_con_ids (con_id);



        /* ====================================================
           6. Признаки договоров

           ВАЖНО:
           Пк2 и От1 храним:
           - отдельно;
           - вместе для существующей бизнес-классификации.
           ==================================================== */

        IF OBJECT_ID('tempdb..#attr_flags') IS NOT NULL
            DROP TABLE #attr_flags;


        ;WITH attr_ranked AS
        (
            SELECT
                  TRY_CAST(a.CON_ID AS bigint) AS con_id


                /* НОВ */
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


                /* Пк2 отдельно */
                , CASE
                      WHEN ISNULL(
                               TRY_CAST(a.[Пк2] AS int),
                               0
                           ) = 1
                          THEN 1
                      ELSE 0
                  END AS is_pk2_flag


                /* От1 отдельно */
                , CASE
                      WHEN ISNULL(
                               TRY_CAST(a.[От1] AS int),
                               0
                           ) = 1
                          THEN 1
                      ELSE 0
                  END AS is_ot1_flag


                /* Старая общая бизнес-категория */
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


            FROM [ALM].[ehd].[attr_DepoFLConditions] a
                 WITH (NOLOCK)

            INNER JOIN #relevant_con_ids r
                ON r.con_id =
                   TRY_CAST(a.CON_ID AS bigint)
        )


        SELECT
              con_id

            , is_nov_flag
            , is_pr2_pr3_flag
            , is_ndp_ndm_flag

            , is_pk2_flag
            , is_ot1_flag
            , is_pk2_ot1_flag

            , is_pk3_pk6_flag
            , is_mpl_flag
            , is_pns_flag

        INTO #attr_flags

        FROM attr_ranked

        WHERE rn = 1;


        CREATE UNIQUE CLUSTERED INDEX
            IX_attr_flags_con
        ON #attr_flags (con_id);



        /* ====================================================
           7. Расчёт и запись месяца
           ==================================================== */

        BEGIN TRY

            BEGIN TRAN;


            IF @ReplaceExisting = 1
            BEGIN

                DELETE
                FROM
                    [alm_report].[depo_fl_client_monthly_stats]

                WHERE
                    observation_month = @ObservationMonth;

            END;



            ;WITH client_flags AS
            (
                /* ============================================
                   Для старых групп сегмент определяется
                   как раньше — по стартовому snapshot.

                   Для новых 03/04, если на старте клиента
                   не было, разрешаем определить ДЧБО
                   по конечному snapshot.
                   ============================================ */

                SELECT
                      c.cli_id

                    , CASE
                          WHEN EXISTS
                          (
                              SELECT 1

                              FROM #bal_base b

                              WHERE
                                  b.cli_id = c.cli_id

                                  AND b.section_name IN
                                  (
                                        N'Срочные'
                                      , N'Накопительный счёт'
                                  )

                                  AND b.TSEGMENTNAME = N'ДЧБО'
                          )
                              THEN N'ДЧБО'


                          WHEN
                              c.client_base_type IN
                              (
                                    N'03. Открывшие вклад'
                                  , N'04. Открывшие НС'
                              )

                              AND EXISTS
                              (
                                  SELECT 1

                                  FROM #bal_end b

                                  WHERE
                                      b.cli_id = c.cli_id

                                      AND b.section_name IN
                                      (
                                            N'Срочные'
                                          , N'Накопительный счёт'
                                      )

                                      AND b.TSEGMENTNAME = N'ДЧБО'
                              )
                              THEN N'ДЧБО'


                          ELSE N'Розница'

                      END AS client_segment

                FROM #client_scope c
            ),



            /* ================================================
               ВКЛАДЫ К ВЫХОДУ
               ================================================ */

            exit_by_con AS
            (
                SELECT
                      b.cli_id
                    , b.con_id

                    , SUM(b.out_rub) AS out_rub


                    /* ФУ */
                    , MAX
                      (
                          CASE
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
                                  THEN 1
                              ELSE 0
                          END
                      ) AS is_fu_flag


                    , MAX(ISNULL(a.is_nov_flag, 0))
                        AS is_nov_flag

                    , MAX(ISNULL(a.is_pr2_pr3_flag, 0))
                        AS is_pr2_pr3_flag

                    , MAX(ISNULL(a.is_ndp_ndm_flag, 0))
                        AS is_ndp_ndm_flag


                    /* Новая детализация */
                    , MAX(ISNULL(a.is_pk2_flag, 0))
                        AS is_pk2_flag

                    , MAX(ISNULL(a.is_ot1_flag, 0))
                        AS is_ot1_flag


                    /* Общая категория сохраняется */
                    , MAX(ISNULL(a.is_pk2_ot1_flag, 0))
                        AS is_pk2_ot1_flag


                    , MAX(ISNULL(a.is_pk3_pk6_flag, 0))
                        AS is_pk3_pk6_flag

                    , MAX(ISNULL(a.is_mpl_flag, 0))
                        AS is_mpl_flag

                    , MAX(ISNULL(a.is_pns_flag, 0))
                        AS is_pns_flag


                FROM #bal_base b

                INNER JOIN #client_scope c
                    ON c.cli_id = b.cli_id

                LEFT JOIN #attr_flags a
                    ON a.con_id = b.con_id

                WHERE
                    b.section_name = N'Срочные'

                    AND b.dt_close_plan >= @ExitFrom
                    AND b.dt_close_plan <= @ExitTo

                GROUP BY
                      b.cli_id
                    , b.con_id
            ),



            /* ================================================
               Старая приоритетная бизнес-классификация
               НЕ МЕНЯЕТСЯ.
               ================================================ */

            exit_classified AS
            (
                SELECT
                      e.cli_id
                    , e.con_id
                    , e.out_rub

                    , e.is_pk2_flag
                    , e.is_ot1_flag


                    , CASE
                          WHEN e.is_fu_flag = 1
                              THEN N'fu'

                          WHEN e.is_nov_flag = 1
                              THEN N'nov'

                          WHEN e.is_pr2_pr3_flag = 1
                              THEN N'pr2_pr3'

                          WHEN e.is_ndp_ndm_flag = 1
                              THEN N'ndp_ndm'

                          WHEN e.is_pk2_ot1_flag = 1
                              THEN N'pk2_ot1'

                          WHEN e.is_pk3_pk6_flag = 1
                              THEN N'pk3_pk6'

                          WHEN e.is_mpl_flag = 1
                              THEN N'mpl'

                          WHEN e.is_pns_flag = 1
                              THEN N'pns'

                          ELSE N'other'

                      END AS exit_category

                FROM exit_by_con e
            ),



            exit_sum AS
            (
                SELECT
                      cli_id

                    , SUM(out_rub)
                        AS exit_td_sum


                    /* ========================================
                       Старые объёмы
                       ======================================== */

                    , SUM(
                          CASE
                              WHEN exit_category = N'fu'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS exit_fu_td_sum


                    , SUM(
                          CASE
                              WHEN exit_category = N'nov'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS exit_nov_td_sum


                    , SUM(
                          CASE
                              WHEN exit_category = N'pr2_pr3'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS exit_pr2_pr3_td_sum


                    , SUM(
                          CASE
                              WHEN exit_category = N'ndp_ndm'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS exit_ndp_ndm_td_sum


                    /* Общая Пк2 / От1 — БЕЗ ИЗМЕНЕНИЙ */
                    , SUM(
                          CASE
                              WHEN exit_category = N'pk2_ot1'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS exit_pk2_ot1_td_sum


                    /* ========================================
                       НОВОЕ: Пк2 отдельно

                       Если стоят одновременно Пк2 и От1,
                       относим договор в Пк2.
                       ======================================== */

                    , SUM(
                          CASE
                              WHEN exit_category = N'pk2_ot1'
                                   AND is_pk2_flag = 1
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS exit_pk2_td_sum


                    /* ========================================
                       НОВОЕ: От1 отдельно

                       Только если на договоре нет Пк2.
                       ======================================== */

                    , SUM(
                          CASE
                              WHEN exit_category = N'pk2_ot1'
                                   AND is_pk2_flag = 0
                                   AND is_ot1_flag = 1
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS exit_ot1_td_sum


                    , SUM(
                          CASE
                              WHEN exit_category = N'pk3_pk6'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS exit_pk3_pk6_td_sum


                    , SUM(
                          CASE
                              WHEN exit_category = N'mpl'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS exit_mpl_td_sum


                    , SUM(
                          CASE
                              WHEN exit_category = N'pns'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS exit_pns_td_sum


                    , SUM(
                          CASE
                              WHEN exit_category = N'other'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS exit_other_td_sum



                    /* ========================================
                       Старые флаги
                       ======================================== */

                    , MAX(
                          CASE
                              WHEN exit_category = N'fu'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_fu_exit_td_flag


                    , MAX(
                          CASE
                              WHEN exit_category = N'nov'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_nov_exit_td_flag


                    , MAX(
                          CASE
                              WHEN exit_category = N'pr2_pr3'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_pr2_pr3_exit_td_flag


                    , MAX(
                          CASE
                              WHEN exit_category = N'ndp_ndm'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_ndp_ndm_exit_td_flag


                    /* Общий старый флаг */
                    , MAX(
                          CASE
                              WHEN exit_category = N'pk2_ot1'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_pk2_ot1_exit_td_flag


                    /* НОВЫЙ Пк2 */
                    , MAX(
                          CASE
                              WHEN exit_category = N'pk2_ot1'
                                   AND is_pk2_flag = 1
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_pk2_exit_td_flag


                    /* НОВЫЙ От1 */
                    , MAX(
                          CASE
                              WHEN exit_category = N'pk2_ot1'
                                   AND is_pk2_flag = 0
                                   AND is_ot1_flag = 1
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_ot1_exit_td_flag


                    , MAX(
                          CASE
                              WHEN exit_category = N'pk3_pk6'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_pk3_pk6_exit_td_flag


                    , MAX(
                          CASE
                              WHEN exit_category = N'mpl'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_mpl_exit_td_flag


                    , MAX(
                          CASE
                              WHEN exit_category = N'pns'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_pns_exit_td_flag


                FROM exit_classified

                GROUP BY
                    cli_id
            ),



            /* ================================================
               Другие вклады вне окна
               ================================================ */

            other_td_flag AS
            (
                SELECT
                      c.cli_id

                    , CASE
                          WHEN EXISTS
                          (
                              SELECT 1

                              FROM #bal_base b

                              WHERE
                                  b.cli_id = c.cli_id

                                  AND b.section_name = N'Срочные'

                                  AND NOT
                                  (
                                          b.dt_close_plan >= @ExitFrom
                                      AND b.dt_close_plan <= @ExitTo
                                  )
                          )
                              THEN 1

                          ELSE 0

                      END AS has_other_rub_td_flag

                FROM #client_scope c
            ),



            /* ================================================
               НС на начало
               ================================================ */

            ns_start AS
            (
                SELECT
                      b.cli_id
                    , SUM(b.out_rub) AS ns_start_sum

                FROM #bal_base b

                INNER JOIN #client_scope c
                    ON c.cli_id = b.cli_id

                WHERE
                    b.section_name = N'Накопительный счёт'

                GROUP BY
                    b.cli_id
            ),



            /* ================================================
               НС на конец
               ================================================ */

            ns_end AS
            (
                SELECT
                      e.cli_id
                    , SUM(e.out_rub) AS ns_end_sum

                FROM #bal_end e

                INNER JOIN #client_scope c
                    ON c.cli_id = e.cli_id

                WHERE
                    e.section_name = N'Накопительный счёт'

                GROUP BY
                    e.cli_id
            ),



            /* ================================================
               SMS
               ================================================ */

            sms_clients AS
            (
                SELECT DISTINCT
                    CAST(m.cli_id AS bigint) AS cli_id

                FROM
                    [ALM].[ehd].[ODS_058_VI_MESSAGE2DWH] m
                    WITH (NOLOCK)

                INNER JOIN #client_scope c
                    ON c.cli_id =
                       CAST(m.cli_id AS bigint)

                WHERE
                    m.msgbegindate >= @OpenFrom

                    AND m.msgbegindate
                        < DATEADD(day, 1, @OpenTo)

                    AND m.cli_id IS NOT NULL
            ),



            /* ================================================
               ОТКРЫТЫЕ ВКЛАДЫ
               ================================================ */

            opened_by_con AS
            (
                SELECT
                      b.cli_id
                    , b.con_id

                    , SUM(b.out_rub) AS out_rub


                    /* ФУ */
                    , MAX
                      (
                          CASE
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
                                  THEN 1
                              ELSE 0
                          END
                      ) AS is_fu_flag


                    , MAX(ISNULL(a.is_nov_flag, 0))
                        AS is_nov_flag

                    , MAX(ISNULL(a.is_pr2_pr3_flag, 0))
                        AS is_pr2_pr3_flag

                    , MAX(ISNULL(a.is_ndp_ndm_flag, 0))
                        AS is_ndp_ndm_flag


                    /* Новая детализация */
                    , MAX(ISNULL(a.is_pk2_flag, 0))
                        AS is_pk2_flag

                    , MAX(ISNULL(a.is_ot1_flag, 0))
                        AS is_ot1_flag


                    /* Общая старая категория */
                    , MAX(ISNULL(a.is_pk2_ot1_flag, 0))
                        AS is_pk2_ot1_flag


                    , MAX(ISNULL(a.is_pk3_pk6_flag, 0))
                        AS is_pk3_pk6_flag

                    , MAX(ISNULL(a.is_mpl_flag, 0))
                        AS is_mpl_flag

                    , MAX(ISNULL(a.is_pns_flag, 0))
                        AS is_pns_flag


                FROM #bal_end b

                INNER JOIN #client_scope c
                    ON c.cli_id = b.cli_id

                LEFT JOIN #attr_flags a
                    ON a.con_id = b.con_id

                WHERE
                    b.section_name = N'Срочные'

                    AND b.dt_open >= @OpenFrom
                    AND b.dt_open <= @OpenTo

                GROUP BY
                      b.cli_id
                    , b.con_id
            ),



            opened_classified AS
            (
                SELECT
                      o.cli_id
                    , o.con_id
                    , o.out_rub

                    , o.is_pk2_flag
                    , o.is_ot1_flag


                    /* Бизнес-приоритет НЕ МЕНЯЕТСЯ */
                    , CASE
                          WHEN o.is_fu_flag = 1
                              THEN N'fu'

                          WHEN o.is_nov_flag = 1
                              THEN N'nov'

                          WHEN o.is_pr2_pr3_flag = 1
                              THEN N'pr2_pr3'

                          WHEN o.is_ndp_ndm_flag = 1
                              THEN N'ndp_ndm'

                          WHEN o.is_pk2_ot1_flag = 1
                              THEN N'pk2_ot1'

                          WHEN o.is_pk3_pk6_flag = 1
                              THEN N'pk3_pk6'

                          WHEN o.is_mpl_flag = 1
                              THEN N'mpl'

                          WHEN o.is_pns_flag = 1
                              THEN N'pns'

                          ELSE N'other'

                      END AS open_category

                FROM opened_by_con o
            ),



            opened_agg AS
            (
                SELECT
                      cli_id


                    /* ========================================
                       Старые объёмы
                       ======================================== */

                    , SUM(
                          CASE
                              WHEN open_category = N'fu'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS opened_fu


                    , SUM(
                          CASE
                              WHEN open_category = N'nov'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS opened_nov


                    , SUM(
                          CASE
                              WHEN open_category = N'pr2_pr3'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS opened_pr2_pr3


                    , SUM(
                          CASE
                              WHEN open_category = N'ndp_ndm'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS opened_ndp_ndm


                    /* Общая категория сохраняется */
                    , SUM(
                          CASE
                              WHEN open_category = N'pk2_ot1'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS opened_pk2_ot1


                    /* НОВОЕ: Пк2 */
                    , SUM(
                          CASE
                              WHEN open_category = N'pk2_ot1'
                                   AND is_pk2_flag = 1
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS opened_pk2


                    /* НОВОЕ: От1 */
                    , SUM(
                          CASE
                              WHEN open_category = N'pk2_ot1'
                                   AND is_pk2_flag = 0
                                   AND is_ot1_flag = 1
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS opened_ot1


                    , SUM(
                          CASE
                              WHEN open_category = N'pk3_pk6'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS opened_pk3_pk6


                    , SUM(
                          CASE
                              WHEN open_category = N'mpl'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS opened_mpl


                    , SUM(
                          CASE
                              WHEN open_category = N'pns'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS opened_pns


                    , SUM(
                          CASE
                              WHEN open_category = N'other'
                                  THEN out_rub
                              ELSE 0
                          END
                      ) AS opened_other


                    , SUM(out_rub)
                        AS opened_total



                    /* ========================================
                       Старые флаги
                       ======================================== */

                    , MAX(
                          CASE
                              WHEN open_category = N'fu'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_opened_fu_flag


                    , MAX(
                          CASE
                              WHEN open_category = N'nov'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_opened_nov_flag


                    , MAX(
                          CASE
                              WHEN open_category = N'pr2_pr3'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_opened_pr2_pr3_flag


                    , MAX(
                          CASE
                              WHEN open_category = N'ndp_ndm'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_opened_ndp_ndm_flag


                    /* Старый общий */
                    , MAX(
                          CASE
                              WHEN open_category = N'pk2_ot1'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_opened_pk2_ot1_flag


                    /* НОВЫЙ Пк2 */
                    , MAX(
                          CASE
                              WHEN open_category = N'pk2_ot1'
                                   AND is_pk2_flag = 1
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_opened_pk2_flag


                    /* НОВЫЙ От1 */
                    , MAX(
                          CASE
                              WHEN open_category = N'pk2_ot1'
                                   AND is_pk2_flag = 0
                                   AND is_ot1_flag = 1
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_opened_ot1_flag


                    , MAX(
                          CASE
                              WHEN open_category = N'pk3_pk6'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_opened_pk3_pk6_flag


                    , MAX(
                          CASE
                              WHEN open_category = N'mpl'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_opened_mpl_flag


                    , MAX(
                          CASE
                              WHEN open_category = N'pns'
                                  THEN 1
                              ELSE 0
                          END
                      ) AS has_opened_pns_flag


                FROM opened_classified

                GROUP BY
                    cli_id
            )



            /* ================================================
               ЗАПИСЬ В ПОСТОЯННУЮ ВИТРИНУ
               ================================================ */

            INSERT INTO
                [alm_report].[depo_fl_client_monthly_stats]
            (
                  observation_month
                , base_date
                , analysis_date
                , is_partial_month

                , cli_id

                , client_base_type
                , segment_flag
                , is_sms_sent
                , base_amount_flag


                /* Выходы */
                , exit_td_sum

                , exit_fu_td_sum
                , exit_nov_td_sum
                , exit_pr2_pr3_td_sum
                , exit_ndp_ndm_td_sum

                , exit_pk2_ot1_td_sum

                /* НОВЫЕ */
                , exit_pk2_td_sum
                , exit_ot1_td_sum

                , exit_pk3_pk6_td_sum
                , exit_mpl_td_sum
                , exit_pns_td_sum
                , exit_other_td_sum


                /* Флаги выходов */
                , has_fu_exit_td_flag
                , has_nov_exit_td_flag
                , has_pr2_pr3_exit_td_flag
                , has_ndp_ndm_exit_td_flag

                , has_pk2_ot1_exit_td_flag

                /* НОВЫЕ */
                , has_pk2_exit_td_flag
                , has_ot1_exit_td_flag

                , has_pk3_pk6_exit_td_flag
                , has_mpl_exit_td_flag
                , has_pns_exit_td_flag

                , has_other_rub_td_flag


                /* НС */
                , ns_start_sum
                , has_ns_gt_1000_flag
                , ns_end_sum
                , ns_delta
                , ns_decrease_flag


                /* Открытия */
                , opened_fu
                , opened_nov
                , opened_pr2_pr3
                , opened_ndp_ndm

                , opened_pk2_ot1

                /* НОВЫЕ */
                , opened_pk2
                , opened_ot1

                , opened_pk3_pk6
                , opened_mpl
                , opened_pns
                , opened_other
                , opened_total


                /* Флаги открытий */
                , has_opened_fu_flag
                , has_opened_nov_flag
                , has_opened_pr2_pr3_flag
                , has_opened_ndp_ndm_flag

                , has_opened_pk2_ot1_flag

                /* НОВЫЕ */
                , has_opened_pk2_flag
                , has_opened_ot1_flag

                , has_opened_pk3_pk6_flag
                , has_opened_mpl_flag
                , has_opened_pns_flag
            )


            SELECT
                  @ObservationMonth
                , @BaseDate
                , @EndDate
                , @IsPartialMonth

                , c.cli_id

                , c.client_base_type
                , f.client_segment


                /* SMS */
                , CASE
                      WHEN sms.cli_id IS NOT NULL
                          THEN 1
                      ELSE 0
                  END


                /* ============================================
                   Бакет

                   Логика старых 01/02 НЕ меняется.

                   Для новых 03/04:
                   поле остаётся "Не определено".
                   Их объёмы описываются соответственно
                   opened_* и ns_end_sum.
                   ============================================ */

                , CASE

                      WHEN
                          c.client_base_type =
                          N'01. Вкладчики к выходу'

                          AND ISNULL(
                                  e.exit_td_sum,
                                  0
                              ) <= 1000000

                          THEN N'1. Выход|НС <= 1.0 млн'


                      WHEN
                          c.client_base_type =
                          N'01. Вкладчики к выходу'

                          AND ISNULL(
                                  e.exit_td_sum,
                                  0
                              ) < 5000000

                          THEN N'2. Выход|НС 1.0-5 млн'


                      WHEN
                          c.client_base_type =
                          N'01. Вкладчики к выходу'

                          THEN N'3. Выход|НС >= 5 млн'


                      WHEN
                          c.client_base_type =
                          N'02. НС без вкладов к выходу'

                          AND ISNULL(
                                  ns1.ns_start_sum,
                                  0
                              ) <= 1000000

                          THEN N'1. Выход|НС <= 1.0 млн'


                      WHEN
                          c.client_base_type =
                          N'02. НС без вкладов к выходу'

                          AND ISNULL(
                                  ns1.ns_start_sum,
                                  0
                              ) < 5000000

                          THEN N'2. Выход|НС 1.0-5 млн'


                      WHEN
                          c.client_base_type =
                          N'02. НС без вкладов к выходу'

                          THEN N'3. Выход|НС >= 5 млн'


                      ELSE N'Не определено'

                  END



                /* ============================================
                   ВКЛАДЫ К ВЫХОДУ
                   ============================================ */

                , ISNULL(e.exit_td_sum, 0)

                , ISNULL(e.exit_fu_td_sum, 0)
                , ISNULL(e.exit_nov_td_sum, 0)
                , ISNULL(e.exit_pr2_pr3_td_sum, 0)
                , ISNULL(e.exit_ndp_ndm_td_sum, 0)


                /* Общая старая категория */
                , ISNULL(e.exit_pk2_ot1_td_sum, 0)

                /* НОВЫЕ */
                , ISNULL(e.exit_pk2_td_sum, 0)
                , ISNULL(e.exit_ot1_td_sum, 0)


                , ISNULL(e.exit_pk3_pk6_td_sum, 0)
                , ISNULL(e.exit_mpl_td_sum, 0)
                , ISNULL(e.exit_pns_td_sum, 0)
                , ISNULL(e.exit_other_td_sum, 0)



                /* ============================================
                   ФЛАГИ ВКЛАДОВ К ВЫХОДУ
                   ============================================ */

                , ISNULL(e.has_fu_exit_td_flag, 0)
                , ISNULL(e.has_nov_exit_td_flag, 0)
                , ISNULL(e.has_pr2_pr3_exit_td_flag, 0)
                , ISNULL(e.has_ndp_ndm_exit_td_flag, 0)


                /* Общий старый */
                , ISNULL(e.has_pk2_ot1_exit_td_flag, 0)

                /* НОВЫЕ */
                , ISNULL(e.has_pk2_exit_td_flag, 0)
                , ISNULL(e.has_ot1_exit_td_flag, 0)


                , ISNULL(e.has_pk3_pk6_exit_td_flag, 0)
                , ISNULL(e.has_mpl_exit_td_flag, 0)
                , ISNULL(e.has_pns_exit_td_flag, 0)

                , ISNULL(ot.has_other_rub_td_flag, 0)



                /* ============================================
                   НС
                   ============================================ */

                , ISNULL(ns1.ns_start_sum, 0)


                , CASE
                      WHEN ISNULL(
                               ns1.ns_start_sum,
                               0
                           ) > 1000

                          THEN 1

                      ELSE 0
                  END


                , ISNULL(ns2.ns_end_sum, 0)


                , ISNULL(ns2.ns_end_sum, 0)
                  - ISNULL(ns1.ns_start_sum, 0)


                , CASE
                      WHEN
                          ISNULL(ns2.ns_end_sum, 0)
                          <
                          ISNULL(ns1.ns_start_sum, 0)

                          THEN 1

                      ELSE 0
                  END



                /* ============================================
                   ОТКРЫТЫЕ ВКЛАДЫ
                   ============================================ */

                , ISNULL(o.opened_fu, 0)
                , ISNULL(o.opened_nov, 0)
                , ISNULL(o.opened_pr2_pr3, 0)
                , ISNULL(o.opened_ndp_ndm, 0)


                /* Общая старая */
                , ISNULL(o.opened_pk2_ot1, 0)

                /* НОВЫЕ */
                , ISNULL(o.opened_pk2, 0)
                , ISNULL(o.opened_ot1, 0)


                , ISNULL(o.opened_pk3_pk6, 0)
                , ISNULL(o.opened_mpl, 0)
                , ISNULL(o.opened_pns, 0)
                , ISNULL(o.opened_other, 0)
                , ISNULL(o.opened_total, 0)



                /* ============================================
                   ФЛАГИ ОТКРЫТЫХ ВКЛАДОВ
                   ============================================ */

                , ISNULL(o.has_opened_fu_flag, 0)
                , ISNULL(o.has_opened_nov_flag, 0)
                , ISNULL(o.has_opened_pr2_pr3_flag, 0)
                , ISNULL(o.has_opened_ndp_ndm_flag, 0)


                /* Общий старый */
                , ISNULL(o.has_opened_pk2_ot1_flag, 0)

                /* НОВЫЕ */
                , ISNULL(o.has_opened_pk2_flag, 0)
                , ISNULL(o.has_opened_ot1_flag, 0)


                , ISNULL(o.has_opened_pk3_pk6_flag, 0)
                , ISNULL(o.has_opened_mpl_flag, 0)
                , ISNULL(o.has_opened_pns_flag, 0)


            FROM #client_scope c

            LEFT JOIN client_flags f
                ON f.cli_id = c.cli_id

            LEFT JOIN sms_clients sms
                ON sms.cli_id = c.cli_id

            LEFT JOIN exit_sum e
                ON e.cli_id = c.cli_id

            LEFT JOIN other_td_flag ot
                ON ot.cli_id = c.cli_id

            LEFT JOIN ns_start ns1
                ON ns1.cli_id = c.cli_id

            LEFT JOIN ns_end ns2
                ON ns2.cli_id = c.cli_id

            LEFT JOIN opened_agg o
                ON o.cli_id = c.cli_id

            OPTION (RECOMPILE);


            COMMIT TRAN;

        END TRY


        BEGIN CATCH

            IF @@TRANCOUNT > 0
                ROLLBACK TRAN;

            THROW;

        END CATCH;



        /* ====================================================
           8. Чистим temp текущего месяца
           ==================================================== */

        DROP TABLE #attr_flags;
        DROP TABLE #relevant_con_ids;
        DROP TABLE #client_scope;



        /* ====================================================
           9. СКОЛЬЖЕНИЕ

           end текущего месяца становится base следующего.
           ==================================================== */

        IF @EndDate < @FinalEndDate
        BEGIN

            TRUNCATE TABLE #bal_base;


            INSERT INTO #bal_base
            (
                  cli_id
                , con_id
                , dt_open
                , dt_close_plan
                , section_name
                , PROD_NAME_res
                , out_rub
                , TSEGMENTNAME
            )

            SELECT
                  cli_id
                , con_id
                , dt_open
                , dt_close_plan
                , section_name
                , PROD_NAME_res
                , out_rub
                , TSEGMENTNAME

            FROM #bal_end;

        END;


        SET @BaseDate = @EndDate;

    END;

END;
GO
```

## В. Пример запуска

Для твоего текущего сценария — база на **31 августа**, статистика сентября по **6 сентября 2026**:

```sql
EXEC [ALM_TEST].[alm_report].[usp_load_depo_fl_client_monthly_stats]

      @StartBaseDate    = '2026-08-31'
    , @FinalEndDate     = '2026-09-06'
    , @ReplaceExisting  = 1;
```

Когда появится полный сентябрь:

```sql
EXEC [ALM_TEST].[alm_report].[usp_load_depo_fl_client_monthly_stats]

      @StartBaseDate    = '2026-08-31'
    , @FinalEndDate     = '2026-09-30'
    , @ReplaceExisting  = 1;
```

А после запуска можно проверить состав новых четырёх групп:

```sql
SELECT
      observation_month
    , client_base_type
    , COUNT(*) AS clients
    , SUM(exit_td_sum) AS exit_td_sum
    , SUM(opened_total) AS opened_total
    , SUM(ns_start_sum) AS ns_start_sum
    , SUM(ns_end_sum) AS ns_end_sum

FROM [ALM_TEST].[alm_report].[depo_fl_client_monthly_stats]

WHERE observation_month = '2026-09-01'

GROUP BY
      observation_month
    , client_base_type

ORDER BY
    client_base_type;
```

Ключевой момент: новая детализация PK2/OT1 сделана **внутри уже присвоенной общей категории `pk2_ot1`**. Поэтому вклад, который из-за более высокого приоритета относится, например, к `nov`, не начнёт внезапно учитываться в `exit_pk2_td_sum` только потому, что у него технически стоит `[Пк2]=1`. Это сохраняет существующую бизнес-логику полностью.
