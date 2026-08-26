Да. Я бы сделал именно так: **постоянная месячная витрина в `[ALM_TEST].[alm_report]` + одна процедура**, которая скользит по месяцам и в `tempdb` одновременно держит только два соседних баланса. Логика полей и классификаций остаётся из твоего текущего скрипта. 

Главное преимущество: при загрузке октября-2025 — августа-2026 баланс на каждую дату читается **один раз**. Например `31.10.2025` сначала является `#bal_end` октября, затем копируется внутри `tempdb` и становится `#bal_base` ноября — повторно из `VW_balance_rest_all` мы его не выгружаем.

## 1. Постоянная таблица

```sql
USE [ALM_TEST];
SET NOCOUNT ON;


/* ============================================================
   Постоянная поклиентная месячная витрина
   ============================================================ */

IF OBJECT_ID(
       N'[alm_report].[depo_fl_client_monthly_stats]',
       N'U'
   ) IS NULL
BEGIN

    CREATE TABLE [alm_report].[depo_fl_client_monthly_stats]
    (
          /* =========================
             Период
             ========================= */

          observation_month date NOT NULL
          -- всегда первое число месяца:
          -- 2026-08-01 = статистика августа

        , base_date date NOT NULL
          -- баланс "было"

        , analysis_date date NOT NULL
          -- фактический баланс "стало"
          -- например 2026-08-24 для неполного августа

        , is_partial_month bit NOT NULL
          -- 1, если analysis_date не конец календарного месяца


          /* =========================
             Клиент
             ========================= */

        , cli_id bigint NOT NULL

        , client_base_type nvarchar(100) NOT NULL
        , segment_flag nvarchar(50) NOT NULL
        , is_sms_sent bit NOT NULL
        , base_amount_flag nvarchar(100) NOT NULL


          /* =========================
             Вклады к выходу — объёмы
             ========================= */

        , exit_td_sum decimal(38,6) NOT NULL

        , exit_fu_td_sum decimal(38,6) NOT NULL
        , exit_nov_td_sum decimal(38,6) NOT NULL
        , exit_pr2_pr3_td_sum decimal(38,6) NOT NULL
        , exit_ndp_ndm_td_sum decimal(38,6) NOT NULL
        , exit_pk2_ot1_td_sum decimal(38,6) NOT NULL
        , exit_pk3_pk6_td_sum decimal(38,6) NOT NULL
        , exit_mpl_td_sum decimal(38,6) NOT NULL
        , exit_pns_td_sum decimal(38,6) NOT NULL
        , exit_other_td_sum decimal(38,6) NOT NULL


          /* =========================
             Вклады к выходу — флаги
             ========================= */

        , has_fu_exit_td_flag bit NOT NULL
        , has_nov_exit_td_flag bit NOT NULL
        , has_pr2_pr3_exit_td_flag bit NOT NULL
        , has_ndp_ndm_exit_td_flag bit NOT NULL
        , has_pk2_ot1_exit_td_flag bit NOT NULL
        , has_pk3_pk6_exit_td_flag bit NOT NULL
        , has_mpl_exit_td_flag bit NOT NULL
        , has_pns_exit_td_flag bit NOT NULL

        , has_other_rub_td_flag bit NOT NULL


          /* =========================
             НС
             ========================= */

        , ns_start_sum decimal(38,6) NOT NULL
        , has_ns_gt_1000_flag bit NOT NULL
        , ns_end_sum decimal(38,6) NOT NULL
        , ns_delta decimal(38,6) NOT NULL
        , ns_decrease_flag bit NOT NULL


          /* =========================
             Открытые вклады — объёмы
             ========================= */

        , opened_fu decimal(38,6) NOT NULL
        , opened_nov decimal(38,6) NOT NULL
        , opened_pr2_pr3 decimal(38,6) NOT NULL
        , opened_ndp_ndm decimal(38,6) NOT NULL
        , opened_pk2_ot1 decimal(38,6) NOT NULL
        , opened_pk3_pk6 decimal(38,6) NOT NULL
        , opened_mpl decimal(38,6) NOT NULL
        , opened_pns decimal(38,6) NOT NULL
        , opened_other decimal(38,6) NOT NULL
        , opened_total decimal(38,6) NOT NULL


          /* =========================
             Открытые вклады — флаги
             ========================= */

        , has_opened_fu_flag bit NOT NULL
        , has_opened_nov_flag bit NOT NULL
        , has_opened_pr2_pr3_flag bit NOT NULL
        , has_opened_ndp_ndm_flag bit NOT NULL
        , has_opened_pk2_ot1_flag bit NOT NULL
        , has_opened_pk3_pk6_flag bit NOT NULL
        , has_opened_mpl_flag bit NOT NULL
        , has_opened_pns_flag bit NOT NULL


          /* =========================
             Техническая информация
             ========================= */

        , load_dt datetime2(0) NOT NULL
              CONSTRAINT DF_depo_fl_client_monthly_stats_load_dt
              DEFAULT SYSDATETIME()


        /* Одна строка клиента на один месяц наблюдения */
        , CONSTRAINT PK_depo_fl_client_monthly_stats
              PRIMARY KEY CLUSTERED
              (
                    observation_month
                  , cli_id
              )
    );


    /* Для просмотра истории конкретного клиента */
    CREATE NONCLUSTERED INDEX
        IX_depo_fl_client_monthly_stats_cli_month
    ON [alm_report].[depo_fl_client_monthly_stats]
    (
          cli_id
        , observation_month
    );

END;
```

---

# 2. Процедура скользящей загрузки

Она принимает:

```text
@StartBaseDate
```

— начальный баланс, например `30.09.2025`;

и:

```text
@FinalEndDate
```

— последнюю доступную дату, например `24.08.2026`.

После этого сама построит:

```text
30.09 → 31.10
31.10 → 30.11
30.11 → 31.12
...
30.06 → 31.07
31.07 → 24.08
```

```sql
USE [ALM_TEST];
SET NOCOUNT ON;
GO


CREATE OR ALTER PROCEDURE
    [alm_report].[usp_load_depo_fl_client_monthly_stats]

      @StartBaseDate  date
    , @FinalEndDate   date
    , @ReplaceExisting bit = 1

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


    /* Начальная дата должна быть концом месяца.
       Последняя дата может быть неполным месяцем. */
    IF @StartBaseDate <> EOMONTH(@StartBaseDate)
    BEGIN
        THROW 51002,
              N'@StartBaseDate должен быть последним календарным днём месяца.',
              1;
    END;


    /* ========================================================
       Исключённые клиенты из текущей версии отчёта
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
       Переменные скользящего периода
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
       1. Первый баланс

       ВАЖНО:
       выгружаем только необходимые поля.
       Это уменьшает размер tempdb.
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


    /* Создаём пустую таблицу той же структуры.
       Она будет постоянно использоваться как следующий snapshot. */

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
           Следующая конечная дата

           Обычно конец следующего месяца.

           Но если конечная дата пользователя раньше,
           например 24.08, берём именно 24.08.
           ==================================================== */

        SET @EndDate =
            EOMONTH(DATEADD(month, 1, @BaseDate));


        IF @EndDate > @FinalEndDate
            SET @EndDate = @FinalEndDate;


        SET @ExitFrom = DATEADD(day, 1, @BaseDate);
        SET @ExitTo   = @EndDate;

        SET @OpenFrom = DATEADD(day, 1, @BaseDate);
        SET @OpenTo   = @EndDate;


        /* Месяц статистики */
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
           3. Загружаем ТОЛЬКО следующий snapshot

           Предыдущий уже находится в #bal_base.

           Поэтому одна дата баланса не читается из ALM дважды.
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
           Если месяц уже загружен и заменять его не хотим —
           расчёты можно пропустить.

           Баланс всё равно понадобится как база следующего месяца.
           ==================================================== */

        IF @ReplaceExisting = 0
           AND EXISTS
           (
               SELECT 1
               FROM [alm_report].[depo_fl_client_monthly_stats]
               WHERE observation_month = @ObservationMonth
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
           4. БАЗА КЛИЕНТОВ ТЕКУЩЕГО МЕСЯЦА
           ==================================================== */

        IF OBJECT_ID('tempdb..#client_scope') IS NOT NULL
            DROP TABLE #client_scope;


        SELECT
              x.cli_id
            , x.client_base_type

        INTO #client_scope

        FROM
        (
            /* ================================================
               01. Вкладчики к выходу
               ================================================ */

            SELECT DISTINCT
                  b.cli_id

                , CAST(
                      N'01. Вкладчики к выходу'
                      AS nvarchar(100)
                  ) AS client_base_type

            FROM #bal_base b

            WHERE
                b.section_name = N'Срочные'

                AND b.dt_close_plan >= @ExitFrom
                AND b.dt_close_plan <= @ExitTo


            UNION ALL


            /* ================================================
               02. НС без вкладов к выходу
               ================================================ */

            SELECT
                  ns.cli_id

                , CAST(
                      N'02. НС без вкладов к выходу'
                      AS nvarchar(100)
                  ) AS client_base_type

            FROM #bal_base ns

            WHERE
                ns.section_name = N'Накопительный счёт'

                AND NOT EXISTS
                (
                    SELECT 1

                    FROM #bal_base td

                    WHERE
                        td.cli_id = ns.cli_id

                        AND td.section_name = N'Срочные'

                        AND td.dt_close_plan >= @ExitFrom
                        AND td.dt_close_plan <= @ExitTo
                )

            GROUP BY
                ns.cli_id

        ) x

        WHERE NOT EXISTS
        (
            SELECT 1
            FROM #excluded_clients ex
            WHERE ex.cli_id = x.cli_id
        );


        CREATE UNIQUE CLUSTERED INDEX
            IX_client_scope_cli
        ON #client_scope (cli_id);



        /* ====================================================
           5. Только нужные con_id

           attr_DepoFLConditions не нужно тащить целиком.
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

           Берём последнюю запись по:
           DT_UPDATE DESC
           loaddate DESC
           ==================================================== */

        IF OBJECT_ID('tempdb..#attr_flags') IS NOT NULL
            DROP TABLE #attr_flags;


        WITH attr_ranked AS
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

           Удаление старого месяца + вставка нового проходят
           одной транзакцией.

           Если расчёт упадёт — старая версия месяца останется.
           ==================================================== */

        BEGIN TRY

            BEGIN TRAN;


            IF @ReplaceExisting = 1
            BEGIN

                DELETE
                FROM [alm_report].[depo_fl_client_monthly_stats]
                WHERE observation_month = @ObservationMonth;

            END;



            ;WITH client_flags AS
            (
                /* ============================================
                   Клиент ДЧБО, если хотя бы один продукт
                   на стартовом балансе помечен ДЧБО
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

                          ELSE N'Розница'
                      END AS client_segment

                FROM #client_scope c
            ),


            /* ================================================
               ВКЛАДЫ К ВЫХОДУ:
               один con_id = одна строка
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
               Приоритет:
               1 ФУ
               2 Нов
               3 Пр2 / Пр3
               4 НДП / НДМ
               5 Пк2 / От1
               6 Пк3 / Пк6
               7 МПЛ
               8 Пнс
               9 остальные
               ================================================ */

            exit_classified AS
            (
                SELECT
                      e.cli_id
                    , e.con_id
                    , e.out_rub

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


                    /* Объёмы */

                    , SUM(
                          CASE WHEN exit_category = N'fu'
                               THEN out_rub ELSE 0 END
                      ) AS exit_fu_td_sum

                    , SUM(
                          CASE WHEN exit_category = N'nov'
                               THEN out_rub ELSE 0 END
                      ) AS exit_nov_td_sum

                    , SUM(
                          CASE WHEN exit_category = N'pr2_pr3'
                               THEN out_rub ELSE 0 END
                      ) AS exit_pr2_pr3_td_sum

                    , SUM(
                          CASE WHEN exit_category = N'ndp_ndm'
                               THEN out_rub ELSE 0 END
                      ) AS exit_ndp_ndm_td_sum

                    , SUM(
                          CASE WHEN exit_category = N'pk2_ot1'
                               THEN out_rub ELSE 0 END
                      ) AS exit_pk2_ot1_td_sum

                    , SUM(
                          CASE WHEN exit_category = N'pk3_pk6'
                               THEN out_rub ELSE 0 END
                      ) AS exit_pk3_pk6_td_sum

                    , SUM(
                          CASE WHEN exit_category = N'mpl'
                               THEN out_rub ELSE 0 END
                      ) AS exit_mpl_td_sum

                    , SUM(
                          CASE WHEN exit_category = N'pns'
                               THEN out_rub ELSE 0 END
                      ) AS exit_pns_td_sum

                    , SUM(
                          CASE WHEN exit_category = N'other'
                               THEN out_rub ELSE 0 END
                      ) AS exit_other_td_sum


                    /* Флаги */

                    , MAX(
                          CASE WHEN exit_category = N'fu'
                               THEN 1 ELSE 0 END
                      ) AS has_fu_exit_td_flag

                    , MAX(
                          CASE WHEN exit_category = N'nov'
                               THEN 1 ELSE 0 END
                      ) AS has_nov_exit_td_flag

                    , MAX(
                          CASE WHEN exit_category = N'pr2_pr3'
                               THEN 1 ELSE 0 END
                      ) AS has_pr2_pr3_exit_td_flag

                    , MAX(
                          CASE WHEN exit_category = N'ndp_ndm'
                               THEN 1 ELSE 0 END
                      ) AS has_ndp_ndm_exit_td_flag

                    , MAX(
                          CASE WHEN exit_category = N'pk2_ot1'
                               THEN 1 ELSE 0 END
                      ) AS has_pk2_ot1_exit_td_flag

                    , MAX(
                          CASE WHEN exit_category = N'pk3_pk6'
                               THEN 1 ELSE 0 END
                      ) AS has_pk3_pk6_exit_td_flag

                    , MAX(
                          CASE WHEN exit_category = N'mpl'
                               THEN 1 ELSE 0 END
                      ) AS has_mpl_exit_td_flag

                    , MAX(
                          CASE WHEN exit_category = N'pns'
                               THEN 1 ELSE 0 END
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
               НС начало
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
               НС конец
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

                FROM [ALM].[ehd].[ODS_058_VI_MESSAGE2DWH] m
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


                    /* Объёмы */

                    , SUM(
                          CASE WHEN open_category = N'fu'
                               THEN out_rub ELSE 0 END
                      ) AS opened_fu

                    , SUM(
                          CASE WHEN open_category = N'nov'
                               THEN out_rub ELSE 0 END
                      ) AS opened_nov

                    , SUM(
                          CASE WHEN open_category = N'pr2_pr3'
                               THEN out_rub ELSE 0 END
                      ) AS opened_pr2_pr3

                    , SUM(
                          CASE WHEN open_category = N'ndp_ndm'
                               THEN out_rub ELSE 0 END
                      ) AS opened_ndp_ndm

                    , SUM(
                          CASE WHEN open_category = N'pk2_ot1'
                               THEN out_rub ELSE 0 END
                      ) AS opened_pk2_ot1

                    , SUM(
                          CASE WHEN open_category = N'pk3_pk6'
                               THEN out_rub ELSE 0 END
                      ) AS opened_pk3_pk6

                    , SUM(
                          CASE WHEN open_category = N'mpl'
                               THEN out_rub ELSE 0 END
                      ) AS opened_mpl

                    , SUM(
                          CASE WHEN open_category = N'pns'
                               THEN out_rub ELSE 0 END
                      ) AS opened_pns

                    , SUM(
                          CASE WHEN open_category = N'other'
                               THEN out_rub ELSE 0 END
                      ) AS opened_other

                    , SUM(out_rub)
                        AS opened_total


                    /* Флаги */

                    , MAX(
                          CASE WHEN open_category = N'fu'
                               THEN 1 ELSE 0 END
                      ) AS has_opened_fu_flag

                    , MAX(
                          CASE WHEN open_category = N'nov'
                               THEN 1 ELSE 0 END
                      ) AS has_opened_nov_flag

                    , MAX(
                          CASE WHEN open_category = N'pr2_pr3'
                               THEN 1 ELSE 0 END
                      ) AS has_opened_pr2_pr3_flag

                    , MAX(
                          CASE WHEN open_category = N'ndp_ndm'
                               THEN 1 ELSE 0 END
                      ) AS has_opened_ndp_ndm_flag

                    , MAX(
                          CASE WHEN open_category = N'pk2_ot1'
                               THEN 1 ELSE 0 END
                      ) AS has_opened_pk2_ot1_flag

                    , MAX(
                          CASE WHEN open_category = N'pk3_pk6'
                               THEN 1 ELSE 0 END
                      ) AS has_opened_pk3_pk6_flag

                    , MAX(
                          CASE WHEN open_category = N'mpl'
                               THEN 1 ELSE 0 END
                      ) AS has_opened_mpl_flag

                    , MAX(
                          CASE WHEN open_category = N'pns'
                               THEN 1 ELSE 0 END
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

                , exit_td_sum

                , exit_fu_td_sum
                , exit_nov_td_sum
                , exit_pr2_pr3_td_sum
                , exit_ndp_ndm_td_sum
                , exit_pk2_ot1_td_sum
                , exit_pk3_pk6_td_sum
                , exit_mpl_td_sum
                , exit_pns_td_sum
                , exit_other_td_sum

                , has_fu_exit_td_flag
                , has_nov_exit_td_flag
                , has_pr2_pr3_exit_td_flag
                , has_ndp_ndm_exit_td_flag
                , has_pk2_ot1_exit_td_flag
                , has_pk3_pk6_exit_td_flag
                , has_mpl_exit_td_flag
                , has_pns_exit_td_flag

                , has_other_rub_td_flag

                , ns_start_sum
                , has_ns_gt_1000_flag
                , ns_end_sum
                , ns_delta
                , ns_decrease_flag

                , opened_fu
                , opened_nov
                , opened_pr2_pr3
                , opened_ndp_ndm
                , opened_pk2_ot1
                , opened_pk3_pk6
                , opened_mpl
                , opened_pns
                , opened_other
                , opened_total

                , has_opened_fu_flag
                , has_opened_nov_flag
                , has_opened_pr2_pr3_flag
                , has_opened_ndp_ndm_flag
                , has_opened_pk2_ot1_flag
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

                , CASE
                      WHEN sms.cli_id IS NOT NULL
                          THEN 1
                      ELSE 0
                  END


                /* ============================================
                   Бакет
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


                /* Вклады к выходу */

                , ISNULL(e.exit_td_sum, 0)

                , ISNULL(e.exit_fu_td_sum, 0)
                , ISNULL(e.exit_nov_td_sum, 0)
                , ISNULL(e.exit_pr2_pr3_td_sum, 0)
                , ISNULL(e.exit_ndp_ndm_td_sum, 0)
                , ISNULL(e.exit_pk2_ot1_td_sum, 0)
                , ISNULL(e.exit_pk3_pk6_td_sum, 0)
                , ISNULL(e.exit_mpl_td_sum, 0)
                , ISNULL(e.exit_pns_td_sum, 0)
                , ISNULL(e.exit_other_td_sum, 0)


                /* Флаги выходов */

                , ISNULL(e.has_fu_exit_td_flag, 0)
                , ISNULL(e.has_nov_exit_td_flag, 0)
                , ISNULL(e.has_pr2_pr3_exit_td_flag, 0)
                , ISNULL(e.has_ndp_ndm_exit_td_flag, 0)
                , ISNULL(e.has_pk2_ot1_exit_td_flag, 0)
                , ISNULL(e.has_pk3_pk6_exit_td_flag, 0)
                , ISNULL(e.has_mpl_exit_td_flag, 0)
                , ISNULL(e.has_pns_exit_td_flag, 0)

                , ISNULL(ot.has_other_rub_td_flag, 0)


                /* НС */

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


                /* Открытия */

                , ISNULL(o.opened_fu, 0)
                , ISNULL(o.opened_nov, 0)
                , ISNULL(o.opened_pr2_pr3, 0)
                , ISNULL(o.opened_ndp_ndm, 0)
                , ISNULL(o.opened_pk2_ot1, 0)
                , ISNULL(o.opened_pk3_pk6, 0)
                , ISNULL(o.opened_mpl, 0)
                , ISNULL(o.opened_pns, 0)
                , ISNULL(o.opened_other, 0)
                , ISNULL(o.opened_total, 0)


                /* Флаги открытий */

                , ISNULL(o.has_opened_fu_flag, 0)
                , ISNULL(o.has_opened_nov_flag, 0)
                , ISNULL(o.has_opened_pr2_pr3_flag, 0)
                , ISNULL(o.has_opened_ndp_ndm_flag, 0)
                , ISNULL(o.has_opened_pk2_ot1_flag, 0)
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
           8. Чистим небольшие temp-объекты месяца
           ==================================================== */

        DROP TABLE #attr_flags;
        DROP TABLE #relevant_con_ids;
        DROP TABLE #client_scope;



        /* ====================================================
           9. СКОЛЬЖЕНИЕ

           #bal_end текущего месяца становится
           #bal_base следующего.

           НИКАКОГО повторного запроса этого snapshot
           в VW_balance_rest_all.
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

## 3. Как запускать всю историю

Чтобы получить статистику **с октября 2025 по 24 августа 2026**:

```sql
EXEC [ALM_TEST].[alm_report].[usp_load_depo_fl_client_monthly_stats]
      @StartBaseDate   = '2025-09-30'
    , @FinalEndDate    = '2026-08-24'
    , @ReplaceExisting = 1;
```

Процедура автоматически сделает:

```text
observation_month    base_date      analysis_date

2025-10-01           2025-09-30     2025-10-31
2025-11-01           2025-10-31     2025-11-30
2025-12-01           2025-11-30     2025-12-31
2026-01-01           2025-12-31     2026-01-31
2026-02-01           2026-01-31     2026-02-28
...
2026-07-01           2026-06-30     2026-07-31
2026-08-01           2026-07-31     2026-08-24
```

Последняя строка будет иметь:

```text
observation_month = 2026-08-01
analysis_date      = 2026-08-24
is_partial_month   = 1
```

### Когда август закроется

Достаточно вызвать:

```sql
EXEC [ALM_TEST].[alm_report].[usp_load_depo_fl_client_monthly_stats]
      @StartBaseDate   = '2026-07-31'
    , @FinalEndDate    = '2026-08-31'
    , @ReplaceExisting = 1;
```

Старый неполный август удалится и заменится полным:

```text
observation_month = 2026-08-01
analysis_date      = 2026-08-31
is_partial_month   = 0
```

То есть дубля августа не возникнет.

### Подключение Excel

После первоначальной загрузки Excel вообще не должен запускать тяжёлую процедуру. Он просто читает готовую таблицу:

```sql
SELECT *
FROM [ALM_TEST].[alm_report].[depo_fl_client_monthly_stats]
ORDER BY
      observation_month
    , cli_id;
```

По нагрузке эта архитектура существенно лучше простого цикла исходного скрипта: в `tempdb` не копится история за 11 месяцев, одновременно живут только **два соседних баланса**, а каждый месячный snapshot исходного баланса читается из `VW_balance_rest_all` только один раз. Маленькие `#client_scope`, `#relevant_con_ids` и `#attr_flags` уничтожаются после каждого месяца.
