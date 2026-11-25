USE ALM_TEST;
SET NOCOUNT ON;

------------------------------------------------------------
-- 0. НАСТРОЙКИ
------------------------------------------------------------

DECLARE @LastWednesday date = '20261118';

-- NULL = все доступные полные недели
-- Можно указать конкретное число, например 4 или 8
DECLARE @Weeks int = NULL;

-- Специальная логика 15% с 1 октября
DECLARE @Promo15From date = '20261001';

-- Маркетплейсы исключаем, как в исходном скрипте
DECLARE @ExcludeMP bit = 1;

DECLARE @ClientBatchSize int = 3000;
DECLARE @BatchPause varchar(8) = '00:00:01';
DECLARE @WeekPause  varchar(8) = '00:00:03';

-- Контрольная сверка первой недели с исходной функцией
DECLARE @VerifyFirstWeek bit = 1;

------------------------------------------------------------
-- ПРОВЕРКА ПАРАМЕТРОВ
------------------------------------------------------------

IF @LastWednesday IS NULL
   OR DATEDIFF(day, '19000103', @LastWednesday) % 7 <> 0
    THROW 50001, N'Последняя дата должна быть средой', 1;

IF @ClientBatchSize IS NULL OR @ClientBatchSize <= 0
    THROW 50002, N'Некорректный размер батча', 1;

IF @@TRANCOUNT > 0 OR (@@OPTIONS & 2) = 2
    THROW 50003, N'Запускайте скрипт вне открытой или неявной транзакции', 1;

------------------------------------------------------------
-- ОПРЕДЕЛЯЕМ МАКСИМАЛЬНО ДОСТУПНОЕ ЧИСЛО НЕДЕЛЬ
------------------------------------------------------------

DECLARE @FirstBaseDate date;

SELECT
    @FirstBaseDate = MIN(CAST(start_date AS date))
FROM ALM_TEST.alm_history.base_rates WITH (NOLOCK)
WHERE [customer segment] = 'private'
OPTION (MAXDOP 1);

IF @FirstBaseDate IS NULL
    THROW 50004, N'История базовых ставок Private отсутствует', 1;

IF DATEADD(day, -6, @LastWednesday) < @FirstBaseDate
    THROW 50005, N'Нет доступных полных недель', 1;

DECLARE @MaxWeeks int =
    1 + DATEDIFF(
        day,
        @FirstBaseDate,
        DATEADD(day, -6, @LastWednesday)
    ) / 7;

IF @Weeks IS NULL
    SET @Weeks = @MaxWeeks;

IF @Weeks NOT BETWEEN 1 AND @MaxWeeks
    THROW 50006, N'Количество недель превышает доступную историю базовых ставок', 1;

------------------------------------------------------------
-- СЛУЖЕБНЫЕ ПЕРЕМЕННЫЕ
------------------------------------------------------------

DECLARE @DateTo date = @LastWednesday;
DECLARE @DateFrom date;

DECLARE @WeekNo int = 1;
DECLARE @BatchNo int;
DECLARE @BatchCount int;

DECLARE @Rows int;
DECLARE @Sec int;
DECLARE @MissingBase int;

DECLARE @FromText varchar(10);
DECLARE @ToText varchar(10);

DECLARE @BaseDateText varchar(10) =
    CONVERT(varchar(10), @FirstBaseDate, 104);

DECLARE @StartTime datetime2 = SYSDATETIME();
DECLARE @WeekTime datetime2;
DECLARE @StepTime datetime2;

RAISERROR(
    N'История базовых ставок начинается с %s. Доступно недель: %d. Выбрано: %d.',
    0, 1, @BaseDateText, @MaxWeeks, @Weeks
) WITH NOWAIT;

------------------------------------------------------------
-- ОЧИСТКА ПОСЛЕ ПРЕДЫДУЩИХ ЗАПУСКОВ
------------------------------------------------------------

DROP TABLE IF EXISTS
    #PR_Result,
    #PR_Joined,
    #PR_Promo,
    #PR_History,
    #PR_Prior,
    #PR_Open,
    #PR_Days,
    #PR_Windows,
    #PR_Allocation,
    #PR_Agg,
    #PR_Raw,
    #PR_Mapped,
    #PR_Reference;

CREATE TABLE #PR_Result
(
    WeekStart       date,
    WeekEnd         date,
    Segment         nvarchar(100),
    SegmentVolume   decimal(18,6),
    SegmentSharePct decimal(18,4),
    ClientRate      decimal(9,6),
    WeightedTerm    decimal(12,2)
);

------------------------------------------------------------
-- ОСНОВНОЙ ЦИКЛ: ОДНА НЕДЕЛЯ = ОДИН БАТЧ
------------------------------------------------------------

WHILE @WeekNo <= @Weeks
BEGIN

    SET @WeekTime = SYSDATETIME();
    SET @DateFrom = DATEADD(day, -6, @DateTo);

    SET @FromText = CONVERT(varchar(10), @DateFrom, 104);
    SET @ToText = CONVERT(varchar(10), @DateTo, 104);

    RAISERROR(
        N'===== PRIVATE: НЕДЕЛЯ %d/%d: %s - %s =====',
        0, 1, @WeekNo, @Weeks, @FromText, @ToText
    ) WITH NOWAIT;

    --------------------------------------------------------
    -- 1. БАЛАНСОВЫЙ СРЕЗ И ОПРЕДЕЛЕНИЕ ПРОМО
    --
    -- Материализуем joined исходной функции.
    -- Витрину, справочник промо и базовые ставки
    -- обрабатываем один раз за неделю.
    --------------------------------------------------------

    SET @StepTime = SYSDATETIME();

    RAISERROR(
        N'1/6 Загрузка балансового среза, промоставок и базовых ставок',
        0, 1
    ) WITH NOWAIT;

    ;WITH mp_list AS
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
            , v.CLI_ID
            , v.OUT_RUB
            , v.termdays
            , v.rate_con
            , v.conv
            , CAST(v.dt_open AS date) AS dt_open

            ------------------------------------------------
            -- Плавающая ставка: строго из PRIVATE_v1
            ------------------------------------------------

            , CASE
                WHEN ISNULL(v.is_floatrate, 0) = 1

                  OR v.PROD_NAME_res LIKE
                     N'Достояние (перем.ст.)%'

                  OR EXISTS
                  (
                      SELECT 1
                      FROM LIQUIDITY.liq.man_FloatContracts fc
                          WITH (NOLOCK)
                      WHERE fc.CON_ID = v.CON_ID
                  )
                    THEN 1
                ELSE 0
              END AS is_float

            ------------------------------------------------
            -- Специальное промо 15% только с 01.10.2026
            ------------------------------------------------

            , CASE
                WHEN v.dt_open >= @Promo15From
                 AND v.termdays BETWEEN 45 AND 70
                 AND ISNULL(v.conv, 'AT_THE_END') = 'AT_THE_END'
                 AND ABS(v.rate_con - 0.1500) <= 0.0005
                    THEN 1
                ELSE 0
              END AS special_15

            ------------------------------------------------
            -- Бакеты для поиска БАЗОВОЙ СТАВКИ.
            -- Важно: 122, 367 и 730 сохранены!
            ------------------------------------------------

            , CASE
                WHEN v.termdays BETWEEN 26 AND 44 THEN 31
                WHEN v.termdays BETWEEN 45 AND 70 THEN 61
                WHEN v.termdays BETWEEN 85 AND 119 THEN 91
                WHEN v.termdays BETWEEN 120 AND 140 THEN 122
                WHEN v.termdays BETWEEN 141 AND 170 THEN 151
                WHEN v.termdays BETWEEN 175 AND 200 THEN 181
                WHEN v.termdays BETWEEN 201 AND 230 THEN 212
                WHEN v.termdays BETWEEN 231 AND 250 THEN 243
                WHEN v.termdays BETWEEN 251 AND 290 THEN 274
                WHEN v.termdays BETWEEN 340 AND 405 THEN 367
                WHEN v.termdays BETWEEN 540 AND 621 THEN 550
                WHEN v.termdays BETWEEN 720 AND 763 THEN 730
                WHEN v.termdays BETWEEN 1090 AND 1140 THEN 1100
                WHEN v.termdays BETWEEN 1450 AND 1475 THEN 1460
                WHEN v.termdays BETWEEN 1795 AND 1830 THEN 1825
                ELSE v.termdays
              END AS base_term

        FROM ALM.ALM.vw_balance_rest_all v WITH (NOLOCK)

        LEFT JOIN mp_list mp
            ON mp.prod_name_res = v.PROD_NAME_res

        WHERE v.DT_REP = @DateTo
          AND v.dt_open >= @DateFrom
          AND v.dt_open < DATEADD(day, 1, @DateTo)

          AND v.section_name = N'Срочные'
          AND v.block_name = N'Привлечение ФЛ'
          AND v.ap = N'Пассив'
          AND v.od_flag = 1
          AND v.cur = '810'
          AND v.OUT_RUB > 0
          AND v.termdays BETWEEN 28 AND 1830

          --------------------------------------------------
          -- Фильтр Private строго из исходной функции
          --------------------------------------------------

          AND
          (
              v.TSegmentName = N'ДЧБО'

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
          )

          AND
          (
              @ExcludeMP = 0
              OR mp.prod_name_res IS NULL
          )
    )

    SELECT
          s.CON_ID
        , s.CLI_ID
        , s.OUT_RUB
        , s.termdays
        , s.rate_con
        , s.dt_open
        , s.is_float

        ----------------------------------------------------
        -- Любое промо из справочника
        -- ИЛИ специальное промо 15%
        ----------------------------------------------------

        , CASE
            WHEN s.special_15 = 1
              OR d.term_bucket IS NOT NULL
                THEN 1
            ELSE 0
          END AS is_promo

        ----------------------------------------------------
        -- Нормализация базовой ставки как в PRIVATE_v1
        ----------------------------------------------------

        , CASE
            WHEN b.base_rate_raw > 1
                THEN b.base_rate_raw / 100.0
            ELSE b.base_rate_raw
          END AS base_rate

    INTO #PR_Joined

    FROM src s

    OUTER APPLY
    (
        SELECT TOP (1)
            d.term_bucket

        FROM ALM_TEST.WORK.promo_new_money_rate_dict d
            WITH (NOLOCK)

        WHERE s.is_float = 0

          AND s.dt_open
              BETWEEN d.date_from AND d.date_to

          AND s.termdays
              BETWEEN d.term_min AND d.term_max

          AND d.conv_type =
              CASE
                  WHEN ISNULL(s.conv, 'AT_THE_END') = 'AT_THE_END'
                      THEN 'AT_THE_END'
                  ELSE 'NOT_AT_THE_END'
              END

          AND s.OUT_RUB
              BETWEEN d.amount_from AND d.amount_to

          AND ABS(s.rate_con - d.promo_rate) <= 0.0005

        ORDER BY d.id DESC
    ) d

    OUTER APPLY
    (
        SELECT TOP (1)
            TRY_CONVERT(decimal(18,8), b.[rate])
                AS base_rate_raw

        FROM ALM_TEST.alm_history.base_rates b
            WITH (NOLOCK)

        WHERE s.is_float = 0

          AND b.[customer segment] = 'private'

          AND TRY_CONVERT(int, b.[term]) = s.base_term

          AND b.[convention] =
              CASE
                  WHEN ISNULL(s.conv, 'AT_THE_END') = 'AT_THE_END'
                      THEN 'at the end'
                  ELSE 'every month'
              END

          AND b.start_date <= s.dt_open

          AND
          (
              b.end_date IS NULL
              OR b.end_date >= s.dt_open
          )

        ORDER BY b.start_date DESC
    ) b

    OPTION (MAXDOP 1, RECOMPILE);

    SET @Rows = @@ROWCOUNT;
    SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

    RAISERROR(
        N'1/6 Готово: %d строк, %d сек.',
        0, 1, @Rows, @Sec
    ) WITH NOWAIT;

    --------------------------------------------------------
    -- Контроль полноты справочника базовых ставок.
    -- Бизнес-логику не меняем, только выводим предупреждение.
    --------------------------------------------------------

    SELECT @MissingBase = COUNT(*)
    FROM #PR_Joined
    WHERE is_float = 0
      AND is_promo = 0
      AND base_rate IS NULL;

    IF @MissingBase > 0
    BEGIN

        RAISERROR(
            N'ВНИМАНИЕ: не найдена базовая ставка у %d непромо-строк. Классификация сохранена как в PRIVATE_v1.',
            0, 1, @MissingBase
        ) WITH NOWAIT;

    END;

    --------------------------------------------------------
    -- 2. ПРОМОВКЛАДЫ И ИСТОРИЯ КЛИЕНТОВ
    --------------------------------------------------------

    SET @StepTime = SYSDATETIME();

    RAISERROR(
        N'2/6 Определение промовкладов и истории клиентов',
        0, 1
    ) WITH NOWAIT;

    SELECT DISTINCT
          CON_ID
        , CLI_ID
        , dt_open

    INTO #PR_Promo

    FROM #PR_Joined

    WHERE is_float = 0
      AND is_promo = 1
      AND CLI_ID IS NOT NULL
      AND CON_ID IS NOT NULL

    OPTION (MAXDOP 1, RECOMPILE);

    SET @Rows = @@ROWCOUNT;

    RAISERROR(
        N'2/6 Найдено промодоговоров: %d',
        0, 1, @Rows
    ) WITH NOWAIT;

    CREATE UNIQUE CLUSTERED INDEX IX_PR_Promo
        ON #PR_Promo(CLI_ID, dt_open, CON_ID)
        WITH (MAXDOP = 1);

    --------------------------------------------------------
    -- Наследуем исходные типы идентификаторов
    --------------------------------------------------------

    SELECT TOP (0)
          h.CLI_ID
        , h.CON_ID
        , CAST(h.DT_OPEN AS date) AS dt_open

    INTO #PR_History

    FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK);

    --------------------------------------------------------
    -- История только клиентов с промовкладами
    --------------------------------------------------------

    IF EXISTS (SELECT 1 FROM #PR_Promo)
    BEGIN

        INSERT INTO #PR_History
        SELECT
              h.CLI_ID
            , h.CON_ID
            , MIN(CAST(h.DT_OPEN AS date))

        FROM LIQUIDITY.liq.DepositContract_all h
            WITH (NOLOCK)

        INNER JOIN
        (
            SELECT DISTINCT CLI_ID
            FROM #PR_Promo
        ) pc
            ON pc.CLI_ID = h.CLI_ID

        WHERE h.DT_OPEN IS NOT NULL
          AND h.DT_OPEN < DATEADD(day, 1, @DateTo)
          AND h.CON_ID IS NOT NULL
          AND h.cli_short_name = N'ФЛ'
          AND h.cur = 'RUR'

          AND
          (
              h.PROD_NAME IS NULL
              OR h.PROD_NAME NOT IN
              (
                  N'Агентские эскроу ФЛ по ставке КС+спред',
                  N'Спец. банк.счёт',
                  N'Залоговый',
                  N'Агентские эскроу ФЛ по ставке КС/2',
                  N'Эскроу',
                  N'Депозит ФЛ (суррогатный договор для счета Новой Афины)'
              )
          )

        GROUP BY h.CLI_ID, h.CON_ID

        OPTION (MAXDOP 1, RECOMPILE);

        SET @Rows = @@ROWCOUNT;

        RAISERROR(
            N'2/6 Загружено исторических договоров: %d',
            0, 1, @Rows
        ) WITH NOWAIT;

    END;

    CREATE UNIQUE CLUSTERED INDEX IX_PR_History
        ON #PR_History(CLI_ID, CON_ID)
        WITH (MAXDOP = 1);

    --------------------------------------------------------
    -- Существующие клиенты: договор до начала недели
    --------------------------------------------------------

    SELECT
          CLI_ID

        , CAST(
            (ROW_NUMBER() OVER (ORDER BY CLI_ID) - 1)
            / @ClientBatchSize + 1
            AS int
          ) AS BatchNo

    INTO #PR_Prior

    FROM #PR_History

    WHERE dt_open < @DateFrom

    GROUP BY CLI_ID

    OPTION (MAXDOP 1, RECOMPILE);

    CREATE UNIQUE CLUSTERED INDEX IX_PR_Prior
        ON #PR_Prior(CLI_ID)
        WITH (MAXDOP = 1);

    CREATE NONCLUSTERED INDEX IX_PR_PriorBatch
        ON #PR_Prior(BatchNo, CLI_ID)
        WITH (MAXDOP = 1);

    SELECT
          @BatchCount = ISNULL(MAX(BatchNo), 0)
        , @Rows = COUNT(*)
    FROM #PR_Prior;

    SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

    RAISERROR(
        N'2/6 Готово: %d существующих клиентов, %d батчей, %d сек.',
        0, 1, @Rows, @BatchCount, @Sec
    ) WITH NOWAIT;

    --------------------------------------------------------
    -- 3. НОВЫЕ ДЕНЬГИ: БАТЧИ ПО КЛИЕНТАМ
    --------------------------------------------------------

    SELECT TOP (0)
          CON_ID
        , CAST(0 AS int) AS new_money_status

    INTO #PR_Allocation

    FROM #PR_Promo;

    SET @BatchNo = 1;

    IF @BatchCount = 0
    BEGIN

        RAISERROR(
            N'3/6 Исторические остатки рассчитывать не требуется',
            0, 1
        ) WITH NOWAIT;

    END;

    WHILE @BatchNo <= @BatchCount
    BEGIN

        SET @StepTime = SYSDATETIME();

        RAISERROR(
            N'3/6 Исторические остатки: батч %d/%d',
            0, 1, @BatchNo, @BatchCount
        ) WITH NOWAIT;

        ----------------------------------------------------
        -- Остаток промовклада на дату открытия
        ----------------------------------------------------

        SELECT
              p.CON_ID
            , p.CLI_ID
            , p.dt_open

            , CAST(
                ISNULL(s.open_balance, 0)
                AS decimal(28,6)
              ) AS open_amount

        INTO #PR_Open

        FROM #PR_Promo p

        INNER JOIN #PR_Prior pc
            ON pc.CLI_ID = p.CLI_ID
           AND pc.BatchNo = @BatchNo

        OUTER APPLY
        (
            SELECT
                SUM(CAST(s.OUT_RUB AS decimal(28,6)))
                    AS open_balance

            FROM LIQUIDITY.liq.DepositContract_Saldo s
                WITH (NOLOCK)

            WHERE s.CON_ID = p.CON_ID

              AND s.DT_FROM < DATEADD(day, 1, p.dt_open)

              AND
              (
                  s.DT_TO IS NULL
                  OR s.DT_TO >= p.dt_open
              )

              AND s.OUT_RUB IS NOT NULL
        ) s

        OPTION (MAXDOP 1, RECOMPILE);

        CREATE UNIQUE CLUSTERED INDEX IX_PR_Open
            ON #PR_Open(CLI_ID, dt_open, CON_ID)
            WITH (MAXDOP = 1);

        ----------------------------------------------------
        -- Общий объём промовкладов за день
        ----------------------------------------------------

        SELECT
              CLI_ID
            , dt_open
            , SUM(open_amount) AS total_open_amount

        INTO #PR_Days

        FROM #PR_Open

        GROUP BY CLI_ID, dt_open

        OPTION (MAXDOP 1, RECOMPILE);

        CREATE UNIQUE CLUSTERED INDEX IX_PR_Days
            ON #PR_Days(CLI_ID, dt_open)
            WITH (MAXDOP = 1);

        ----------------------------------------------------
        -- Окна анализа исторических остатков
        ----------------------------------------------------

        SELECT
              CLI_ID

            , DATEFROMPARTS(
                  YEAR(dt_open),
                  MONTH(dt_open),
                  1
              ) AS month_start

            , DATEADD(
                  month, -1,
                  DATEFROMPARTS(
                      YEAR(dt_open),
                      MONTH(dt_open),
                      1
                  )
              ) AS prev_start

            , MAX(dt_open) AS max_open

        INTO #PR_Windows

        FROM #PR_Days

        GROUP BY
              CLI_ID
            , YEAR(dt_open)
            , MONTH(dt_open)

        OPTION (MAXDOP 1, RECOMPILE);

        ----------------------------------------------------
        -- Событийный расчёт остатков
        -- Строго по исходной функции PRIVATE_v1
        ----------------------------------------------------

        ;WITH saldo_clipped AS
        (
            SELECT
                  w.CLI_ID
                , w.month_start
                , w.prev_start
                , w.max_open

                , CAST(s.OUT_RUB AS decimal(28,6))
                    AS out_rub

                , bounds.effective_from
                , bounds.effective_to

            FROM #PR_Windows w

            INNER JOIN #PR_History h
                ON h.CLI_ID = w.CLI_ID
               AND h.dt_open <= w.max_open

            INNER JOIN LIQUIDITY.liq.DepositContract_Saldo s
                WITH (NOLOCK)

                ON s.CON_ID = h.CON_ID

               AND s.DT_FROM < DATEADD(day, 1, w.max_open)

               AND
               (
                   s.DT_TO IS NULL
                   OR s.DT_TO >= w.prev_start
               )

            CROSS APPLY
            (
                SELECT
                    (
                        SELECT MAX(v.dt)
                        FROM (VALUES
                              (CAST(s.DT_FROM AS date))
                            , (h.dt_open)
                            , (w.prev_start)
                        ) v(dt)
                    ) AS effective_from

                    , (
                        SELECT MIN(v.dt)
                        FROM (VALUES
                              (
                                  ISNULL(
                                      CAST(s.DT_TO AS date),
                                      w.max_open
                                  )
                              )
                            , (w.max_open)
                        ) v(dt)
                    ) AS effective_to
            ) bounds

            WHERE s.OUT_RUB IS NOT NULL
              AND bounds.effective_from <= bounds.effective_to
        ),

        saldo_events AS
        (
            SELECT
                  s.CLI_ID
                , s.month_start
                , e.event_date
                , e.delta

            FROM saldo_clipped s

            CROSS APPLY
            (
                VALUES
                (
                    s.effective_from,
                    s.out_rub
                ),
                (
                    CASE
                        WHEN s.effective_to < s.max_open
                            THEN DATEADD(day, 1, s.effective_to)
                        ELSE NULL
                    END,
                    -s.out_rub
                )
            ) e(event_date, delta)

            WHERE e.event_date IS NOT NULL

            UNION ALL

            SELECT
                  CLI_ID
                , month_start
                , prev_start
                , CAST(0 AS decimal(28,6))
            FROM #PR_Windows

            UNION ALL

            SELECT
                  CLI_ID
                , month_start
                , month_start
                , CAST(0 AS decimal(28,6))
            FROM #PR_Windows

            UNION ALL

            SELECT
                  CLI_ID
                , DATEFROMPARTS(
                      YEAR(dt_open),
                      MONTH(dt_open),
                      1
                  )
                , dt_open
                , CAST(0 AS decimal(28,6))
            FROM #PR_Days
        ),

        event_agg AS
        (
            SELECT
                  CLI_ID
                , month_start
                , event_date
                , SUM(delta) AS delta

            FROM saldo_events

            GROUP BY
                  CLI_ID
                , month_start
                , event_date
        ),

        timeline AS
        (
            SELECT
                  CLI_ID
                , month_start
                , event_date

                , SUM(delta) OVER
                  (
                      PARTITION BY CLI_ID, month_start
                      ORDER BY event_date

                      ROWS BETWEEN UNBOUNDED PRECEDING
                               AND CURRENT ROW
                  ) AS balance

            FROM event_agg
        ),

        timeline_stats AS
        (
            SELECT
                  CLI_ID
                , month_start
                , event_date
                , balance

                , MAX(
                    CASE
                        WHEN event_date < month_start
                            THEN balance
                    END
                  ) OVER
                  (
                      PARTITION BY CLI_ID, month_start
                  ) AS prev_month_peak

                , MAX(
                    CASE
                        WHEN event_date >= month_start
                            THEN balance
                    END
                  ) OVER
                  (
                      PARTITION BY CLI_ID, month_start
                      ORDER BY event_date

                      ROWS BETWEEN UNBOUNDED PRECEDING
                               AND 1 PRECEDING
                  ) AS current_month_peak_before

            FROM timeline
        ),

        day_balances AS
        (
            SELECT
                  d.CLI_ID
                , d.dt_open
                , d.total_open_amount
                , ISNULL(t.balance, 0) AS balance_today

                , CASE
                    WHEN ISNULL(t.prev_month_peak, 0)
                       >= ISNULL(t.current_month_peak_before, 0)
                        THEN ISNULL(t.prev_month_peak, 0)

                    ELSE ISNULL(t.current_month_peak_before, 0)
                  END AS comparison_peak

            FROM #PR_Days d

            LEFT JOIN timeline_stats t
                ON t.CLI_ID = d.CLI_ID

               AND t.month_start = DATEFROMPARTS(
                      YEAR(d.dt_open),
                      MONTH(d.dt_open),
                      1
                   )

               AND t.event_date = d.dt_open
        ),

        day_budget AS
        (
            SELECT
                  CLI_ID
                , dt_open
                , total_open_amount

                , CASE
                    WHEN total_open_amount <= 0 THEN 0

                    WHEN balance_today <= comparison_peak
                        THEN 0

                    WHEN balance_today - comparison_peak
                         >= total_open_amount
                        THEN total_open_amount

                    ELSE balance_today - comparison_peak
                  END AS new_money_budget

            FROM day_balances
        )

        ----------------------------------------------------
        -- Статусы новых денег строго из PRIVATE_v1
        ----------------------------------------------------

        INSERT INTO #PR_Allocation

        SELECT
              p.CON_ID

            , CASE
                WHEN p.open_amount <= 0
                  OR d.total_open_amount <= 0
                    THEN 0

                WHEN d.new_money_budget
                     >= d.total_open_amount - 0.01
                    THEN 2

                WHEN d.new_money_budget > 0
                    THEN 1

                ELSE 0
              END AS new_money_status

        FROM #PR_Open p

        INNER JOIN day_budget d
            ON d.CLI_ID = p.CLI_ID
           AND d.dt_open = p.dt_open

        OPTION (MAXDOP 1, RECOMPILE);

        SET @Rows = @@ROWCOUNT;

        DROP TABLE
            #PR_Open,
            #PR_Days,
            #PR_Windows;

        SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

        RAISERROR(
            N'3/6 Батч %d/%d завершён: %d строк, %d сек.',
            0, 1, @BatchNo, @BatchCount, @Rows, @Sec
        ) WITH NOWAIT;

        IF @BatchNo < @BatchCount
            WAITFOR DELAY @BatchPause;

        SET @BatchNo = @BatchNo + 1;

    END;

    CREATE NONCLUSTERED INDEX IX_PR_Allocation
        ON #PR_Allocation(CON_ID)
        INCLUDE (new_money_status)
        WITH (MAXDOP = 1);

    DROP TABLE #PR_History, #PR_Promo;

    --------------------------------------------------------
    -- 4. КЛАССИФИКАЦИЯ И АГРЕГАЦИЯ ПО БАКЕТАМ
    --------------------------------------------------------

    SET @StepTime = SYSDATETIME();

    RAISERROR(
        N'4/6 Классификация Private и агрегация по бакетам',
        0, 1
    ) WITH NOWAIT;

    ;WITH classified AS
    (
        SELECT
              j.OUT_RUB
            , j.rate_con

            ------------------------------------------------
            -- Порядок классификации строго из PRIVATE_v1
            ------------------------------------------------

            , CASE
                WHEN j.is_float = 1
                    THEN N'Плавающая ставка'

                WHEN j.is_promo = 1
                 AND j.CLI_ID IS NOT NULL
                 AND pc.CLI_ID IS NULL
                    THEN N'Новые клиенты'

                WHEN j.is_promo = 1
                 AND a.new_money_status = 2
                    THEN N'Новые деньги'

                WHEN j.is_promo = 1
                 AND a.new_money_status = 1
                    THEN N'Частично новые деньги'

                WHEN j.is_promo = 1
                    THEN N'Промоставка на действующие деньги'

                WHEN j.rate_con > j.base_rate + 0.0000005
                    THEN N'Выше базовой ставки'

                ELSE N'Базовая ставка или ниже'
              END AS segment_name

            ------------------------------------------------
            -- ОТЧЁТНЫЕ бакеты.
            -- Именно по ним считаем средневзвешенный срок.
            ------------------------------------------------

            , CASE
                WHEN j.termdays BETWEEN 26 AND 44 THEN 31
                WHEN j.termdays BETWEEN 45 AND 70 THEN 61
                WHEN j.termdays BETWEEN 85 AND 119 THEN 91
                WHEN j.termdays BETWEEN 120 AND 140 THEN 124
                WHEN j.termdays BETWEEN 141 AND 170 THEN 151
                WHEN j.termdays BETWEEN 175 AND 200 THEN 181
                WHEN j.termdays BETWEEN 201 AND 230 THEN 212
                WHEN j.termdays BETWEEN 231 AND 250 THEN 243
                WHEN j.termdays BETWEEN 251 AND 290 THEN 274
                WHEN j.termdays BETWEEN 340 AND 405 THEN 365
                WHEN j.termdays BETWEEN 540 AND 621 THEN 550
                WHEN j.termdays BETWEEN 720 AND 763 THEN 750
                WHEN j.termdays BETWEEN 1090 AND 1140 THEN 1100
                WHEN j.termdays BETWEEN 1450 AND 1475 THEN 1460
                WHEN j.termdays BETWEEN 1795 AND 1830 THEN 1825
                ELSE j.termdays
              END AS bucket_code

        FROM #PR_Joined j

        LEFT JOIN #PR_Prior pc
            ON pc.CLI_ID = j.CLI_ID

        -- Сохраняем ключ соединения исходной функции
        LEFT JOIN #PR_Allocation a
            ON a.CON_ID = j.CON_ID
    )

    --------------------------------------------------------
    -- Материализуем агрегацию, чтобы не считать её повторно
    --------------------------------------------------------

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

        ----------------------------------------------------
        -- ВАЖНО: в Private допускаются NULL-ставки.
        -- Расчёт взвешенной ставки строго из PRIVATE_v1.
        ----------------------------------------------------

        , SUM(
            CASE
                WHEN rate_con IS NOT NULL
                    THEN CAST(OUT_RUB AS decimal(24,2))
                       * CAST(rate_con AS decimal(12,8))
            END
          ) AS weighted_rate_sum

        , SUM(
            CASE
                WHEN rate_con IS NOT NULL
                    THEN CAST(OUT_RUB AS decimal(24,2))
                ELSE 0
            END
          ) AS rate_volume

    INTO #PR_Agg

    FROM classified

    GROUP BY GROUPING SETS
    (
          (bucket_code, segment_name)
        , (bucket_code)
    )

    OPTION (MAXDOP 1, RECOMPILE);

    DROP TABLE
        #PR_Joined,
        #PR_Prior,
        #PR_Allocation;

    --------------------------------------------------------
    -- ВОСПРОИЗВОДИМ ВЫХОД ИСХОДНОЙ PRIVATE_v1
    -- С исходной точностью округлений
    --------------------------------------------------------

    SELECT
          a.bucket_code AS Bucket
        , a.segment_name AS Segment

        , CAST(
            a.segment_vol / 1000000000.0
            AS decimal(18,6)
          ) AS SegmentVolume

        , CAST(
            a.segment_vol * 1.0
            / NULLIF(t.grand_total, 0)
            AS decimal(18,4)
          ) AS SegmentSharePct

        , CAST(
            a.weighted_rate_sum
            / NULLIF(a.rate_volume, 0)
            AS decimal(9,6)
          ) AS ClientRate

    INTO #PR_Raw

    FROM #PR_Agg a

    CROSS JOIN
    (
        SELECT SUM(segment_vol) AS grand_total
        FROM #PR_Agg
        WHERE is_total = 0
    ) t

    OPTION (MAXDOP 1, RECOMPILE);

    DROP TABLE #PR_Agg;

    SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

    RAISERROR(
        N'4/6 Классификация и расчёт бакетов завершены: %d сек.',
        0, 1, @Sec
    ) WITH NOWAIT;

    SET @Sec = DATEDIFF(second, @WeekTime, SYSDATETIME());

    RAISERROR(
        N'Оптимизированный расчёт недели: %d сек.',
        0, 1, @Sec
    ) WITH NOWAIT;

    --------------------------------------------------------
    -- 5. КОНТРОЛЬНАЯ СВЕРКА С PRIVATE_v1
    --
    -- Только первая неделя и только после 01.10.2026,
    -- поскольку ранее специальная логика 15% изменена.
    --------------------------------------------------------

    IF @VerifyFirstWeek = 1
       AND @WeekNo = 1
       AND @DateFrom >= @Promo15From
    BEGIN

        SET @StepTime = SYSDATETIME();

        RAISERROR(
            N'5/6 Контрольный запуск исходной PRIVATE_v1',
            0, 1
        ) WITH NOWAIT;

        SELECT
              Bucket
            , Segment
            , SegmentVolume
            , SegmentSharePct
            , ClientRate

        INTO #PR_Reference

        FROM ALM_TEST.reports.
            [fn_NewAttractionVolumes_forDimaPRIVATE_v1]
        (
              @DateTo
            , @DateFrom
            , @DateTo
            , @ExcludeMP
        )

        OPTION (MAXDOP 1, RECOMPILE);

        IF EXISTS
        (
            SELECT * FROM #PR_Raw

            EXCEPT

            SELECT * FROM #PR_Reference
        )
        OR EXISTS
        (
            SELECT * FROM #PR_Reference

            EXCEPT

            SELECT * FROM #PR_Raw
        )
        BEGIN

            SELECT
                  N'Оптимизированный' AS SourceVersion
                , d.*

            FROM
            (
                SELECT * FROM #PR_Raw
                EXCEPT
                SELECT * FROM #PR_Reference
            ) d

            UNION ALL

            SELECT
                  N'Исходная PRIVATE_v1'
                , d.*

            FROM
            (
                SELECT * FROM #PR_Reference
                EXCEPT
                SELECT * FROM #PR_Raw
            ) d

            ORDER BY Bucket, Segment, SourceVersion;

            THROW 50007,
                N'Обнаружено расхождение с PRIVATE_v1. Расчёт остановлен.',
                1;

        END;

        DROP TABLE #PR_Reference;

        SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

        RAISERROR(
            N'5/6 ПРОВЕРКА ПРОЙДЕНА. Совпадение с PRIVATE_v1. Время исходной функции: %d сек.',
            0, 1, @Sec
        ) WITH NOWAIT;

    END
    ELSE
    BEGIN

        RAISERROR(
            N'5/6 Контрольная сверка пропущена',
            0, 1
        ) WITH NOWAIT;

    END;

    --------------------------------------------------------
    -- 6. ВОСПРОИЗВОДИМ СКРИПТ КОЛЛЕГ ДЛЯ PRIVATE
    --------------------------------------------------------

    SET @StepTime = SYSDATETIME();

    ;WITH mapped AS
    (
        SELECT
              Bucket

            , CASE
                WHEN Segment = N'Базовая ставка или ниже'
                    THEN N'Базовая ставка'

                WHEN Segment = N'Выше базовой ставки'
                    THEN N'Индивидуальные согласования'

                WHEN Segment = N'Промоставка на действующие деньги'
                    THEN N'Удержание'

                WHEN Segment IN
                (
                    N'Новые деньги',
                    N'Частично новые деньги'
                )
                    THEN N'Новые деньги'

                ELSE Segment
              END AS Segment

            , SegmentVolume
            , SegmentSharePct
            , ClientRate

        FROM #PR_Raw
    )

    --------------------------------------------------------
    -- Исходная выгрузка коллег:
    -- бакеты + укрупнённые группы
    --------------------------------------------------------

    SELECT
          Bucket
        , Segment

        , CAST(
            SUM(SegmentVolume)
            AS decimal(18,6)
          ) AS SegmentVolume

        , CAST(
            SUM(SegmentSharePct)
            AS decimal(18,4)
          ) AS SegmentSharePct

        , CAST(
            CASE
                WHEN COUNT(*) = 1
                    THEN MAX(ClientRate)

                ELSE
                    SUM(SegmentVolume * ClientRate)
                    / NULLIF(SUM(SegmentVolume), 0)
            END
            AS decimal(9,6)
          ) AS ClientRate

    INTO #PR_Mapped

    FROM mapped

    GROUP BY Bucket, Segment

    OPTION (MAXDOP 1, RECOMPILE);

    --------------------------------------------------------
    -- ФИНАЛЬНЫЕ ЧЕТЫРЕ ПОКАЗАТЕЛЯ
    -- Аналог расчётов коллег в Excel
    --------------------------------------------------------

    INSERT INTO #PR_Result
    SELECT
          @DateFrom
        , @DateTo
        , Segment

        ----------------------------------------------------
        -- А. Объём, млрд руб.
        ----------------------------------------------------

        , CAST(
            SUM(SegmentVolume)
            AS decimal(18,6)
          )

        ----------------------------------------------------
        -- Б. Доля от общего объёма
        ----------------------------------------------------

        , CAST(
            SUM(SegmentSharePct)
            AS decimal(18,4)
          )

        ----------------------------------------------------
        -- В. Средневзвешенная ставка
        -- СУММПРОИЗВ / СУММ
        ----------------------------------------------------

        , CAST(
            CAST(
                COALESCE(
                    SUM(SegmentVolume * ClientRate), 0
                )
                AS decimal(30,12)
            )
            / NULLIF(
                CAST(SUM(SegmentVolume) AS decimal(28,6)),
                0
            )
            AS decimal(9,6)
          )

        ----------------------------------------------------
        -- Г. Средневзвешенная срочность ПО БАКЕТАМ
        ----------------------------------------------------

        , CAST(
            CAST(
                SUM(SegmentVolume * Bucket)
                AS decimal(30,10)
            )
            / NULLIF(
                CAST(SUM(SegmentVolume) AS decimal(28,6)),
                0
            )
            AS decimal(12,2)
          )

    FROM #PR_Mapped

    GROUP BY Segment

    OPTION (MAXDOP 1, RECOMPILE);

    SET @Rows = @@ROWCOUNT;

    IF @Rows = 0
    BEGIN

        RAISERROR(
            N'ВНИМАНИЕ: за текущую неделю нет данных.',
            0, 1
        ) WITH NOWAIT;

    END;

    DROP TABLE
        #PR_Raw,
        #PR_Mapped;

    SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

    RAISERROR(
        N'6/6 Итоговая агрегация: %d строк, %d сек.',
        0, 1, @Rows, @Sec
    ) WITH NOWAIT;

    SET @Sec = DATEDIFF(second, @WeekTime, SYSDATETIME());

    RAISERROR(
        N'===== PRIVATE: НЕДЕЛЯ %d/%d ЗАВЕРШЕНА: %d сек. =====',
        0, 1, @WeekNo, @Weeks, @Sec
    ) WITH NOWAIT;

    --------------------------------------------------------
    -- Пауза и переход к следующей неделе
    --------------------------------------------------------

    IF @WeekNo < @Weeks
        WAITFOR DELAY @WeekPause;

    SET @DateTo = DATEADD(day, -7, @DateTo);
    SET @WeekNo = @WeekNo + 1;

END;

------------------------------------------------------------
-- ИТОГОВАЯ ВЫГРУЗКА
------------------------------------------------------------

SET @Sec = DATEDIFF(second, @StartTime, SYSDATETIME());

RAISERROR(
    N'PRIVATE: РАСЧЁТ ЗАВЕРШЁН. Общее время: %d сек.',
    0, 1, @Sec
) WITH NOWAIT;

SELECT
      WeekStart
    , WeekEnd
    , Segment
    , SegmentVolume
    , SegmentSharePct
    , ClientRate
    , WeightedTerm

FROM #PR_Result

ORDER BY
    WeekEnd DESC,

    CASE Segment
        WHEN N'Базовая ставка' THEN 1
        WHEN N'Плавающая ставка' THEN 2
        WHEN N'Удержание' THEN 3
        WHEN N'Новые деньги' THEN 4
        WHEN N'Новые клиенты' THEN 5
        WHEN N'Индивидуальные согласования' THEN 6
        WHEN N'Итого' THEN 99
        ELSE 98
    END;


    
