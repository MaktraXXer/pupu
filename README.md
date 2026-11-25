USE ALM_TEST;
SET NOCOUNT ON;

------------------------------------------------------------
-- 0. НАСТРОЙКИ
------------------------------------------------------------

DECLARE @AsOfDate date = '20261125';
DECLARE @Weeks int = 12;
DECLARE @DataLag int = 2;

-- Специальная классификация 15% только с 01.10.2026
DECLARE @Promo15From date = '20261001';

-- Размер клиентского батча
DECLARE @ClientBatchSize int = 3000;

-- Паузы для снижения непрерывной нагрузки
DECLARE @BatchPause varchar(8) = '00:00:01';
DECLARE @WeekPause  varchar(8) = '00:00:03';

-- 1 = проверить первую неделю по исходной v1.6
-- 0 = выполнять только оптимизированный расчёт
DECLARE @VerifyFirstWeek bit = 1;

IF @Weeks IS NULL
   OR @Weeks NOT BETWEEN 1 AND 12
   OR @ClientBatchSize IS NULL
   OR @ClientBatchSize <= 0
    THROW 50001, N'Некорректные параметры', 1;

DECLARE @AvailableDate date =
    DATEADD(day, -@DataLag, @AsOfDate);

-- Последняя доступная среда
DECLARE @DateTo date =
    DATEADD(
        day,
        -(DATEDIFF(day, '19000103', @AvailableDate) % 7),
        @AvailableDate
    );

DECLARE @DateFrom date;
DECLARE @WeekNo int = 1;
DECLARE @BatchNo int;
DECLARE @BatchCount int;

DECLARE @Rows int;
DECLARE @Sec int;

DECLARE @FromText varchar(10);
DECLARE @ToText varchar(10);

DECLARE @StartTime datetime2 = SYSDATETIME();
DECLARE @WeekTime datetime2;
DECLARE @StepTime datetime2;

------------------------------------------------------------
-- ОЧИСТКА ПОСЛЕ ПРЕДЫДУЩИХ ЗАПУСКОВ
------------------------------------------------------------

DROP TABLE IF EXISTS
    #W3_Result,
    #W3_Src,
    #W3_Special,
    #W3_History,
    #W3_Prior,
    #W3_Open,
    #W3_Days,
    #W3_Windows,
    #W3_Allocation,
    #W3_Agg,
    #W3_Raw,
    #W3_Mapped,
    #W3_Reference;

CREATE TABLE #W3_Result
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
-- ОСНОВНОЙ ЦИКЛ ПО НЕДЕЛЯМ
------------------------------------------------------------

WHILE @WeekNo <= @Weeks
BEGIN

    SET @WeekTime = SYSDATETIME();
    SET @DateFrom = DATEADD(day, -6, @DateTo);

    SET @FromText = CONVERT(varchar(10), @DateFrom, 104);
    SET @ToText = CONVERT(varchar(10), @DateTo, 104);

    RAISERROR(
        N'===== НЕДЕЛЯ %d/%d: %s - %s =====',
        0, 1,
        @WeekNo, @Weeks, @FromText, @ToText
    ) WITH NOWAIT;

    IF @DateTo < @Promo15From
    BEGIN
        RAISERROR(
            N'Специальная классификация 15%% отключена: период до 01.10.2026',
            0, 1
        ) WITH NOWAIT;
    END
    ELSE
    BEGIN
        RAISERROR(
            N'Специальная классификация 15%% включена для вкладов с 01.10.2026',
            0, 1
        ) WITH NOWAIT;
    END;

    --------------------------------------------------------
    -- 1. БАЛАНСОВЫЙ СРЕЗ
    -- Фильтры исходной v1.6
    --------------------------------------------------------

    SET @StepTime = SYSDATETIME();

    RAISERROR(
        N'1/6 Загрузка балансового среза',
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
    )

    SELECT
          v.CON_ID
        , v.CLI_ID
        , v.OUT_RUB
        , v.PROD_NAME_res
        , v.termdays
        , v.rate_con
        , v.conv
        , CAST(v.dt_open AS date) AS dt_open

        , CASE
            WHEN mp.prod_name_res IS NOT NULL THEN 1
            ELSE 0
          END AS is_mp

        ----------------------------------------------------
        -- НОВОЕ УСЛОВИЕ:
        -- специальная логика 15% только с 01.10.2026
        ----------------------------------------------------

        , CASE
            WHEN v.dt_open >= @Promo15From
             AND v.termdays BETWEEN 45 AND 70
             AND ISNULL(v.conv, 'AT_THE_END') = 'AT_THE_END'
             AND ABS(v.rate_con - 0.1500) <= 0.0005
                THEN 1
            ELSE 0
          END AS special_2m

    INTO #W3_Src

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
      AND ISNULL(v.is_floatrate, 0) = 0
      AND v.rate_con IS NOT NULL
      AND v.termdays BETWEEN 28 AND 1830

      AND NOT EXISTS
      (
          SELECT 1
          FROM LIQUIDITY.liq.man_FloatContracts fc
              WITH (NOLOCK)
          WHERE fc.CON_ID = v.CON_ID
      )

      AND
      (
          v.TSegmentName IS NULL
          OR v.TSegmentName <> N'ДЧБО'
      )

      AND
      (
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

    OPTION (MAXDOP 1, RECOMPILE);

    SET @Rows = @@ROWCOUNT;
    SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

    RAISERROR(
        N'1/6 Готово: %d строк, %d сек.',
        0, 1, @Rows, @Sec
    ) WITH NOWAIT;

    --------------------------------------------------------
    -- 2. СПЕЦИАЛЬНЫЕ ВКЛАДЫ 15% И ИСТОРИЯ КЛИЕНТОВ
    --------------------------------------------------------

    SET @StepTime = SYSDATETIME();

    RAISERROR(
        N'2/6 Определение клиентов специального промо 15%%',
        0, 1
    ) WITH NOWAIT;

    SELECT DISTINCT
          CON_ID
        , CLI_ID
        , dt_open

    INTO #W3_Special

    FROM #W3_Src

    WHERE special_2m = 1
      AND is_mp = 0

      AND
      (
          PROD_NAME_res IS NULL
          OR PROD_NAME_res <> N'Активный процент'
      )

      AND CLI_ID IS NOT NULL
      AND CON_ID IS NOT NULL

    OPTION (MAXDOP 1, RECOMPILE);

    SET @Rows = @@ROWCOUNT;

    RAISERROR(
        N'2/6 Найдено специальных договоров: %d',
        0, 1, @Rows
    ) WITH NOWAIT;

    CREATE UNIQUE CLUSTERED INDEX IX_Special
        ON #W3_Special(CLI_ID, dt_open, CON_ID)
        WITH (MAXDOP = 1);

    --------------------------------------------------------
    -- Наследуем типы идентификаторов из источника
    --------------------------------------------------------

    SELECT TOP (0)
          h.CLI_ID
        , h.CON_ID
        , CAST(h.DT_OPEN AS date) AS dt_open

    INTO #W3_History

    FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK);

    --------------------------------------------------------
    -- Если специальных договоров нет,
    -- историю договоров не загружаем
    --------------------------------------------------------

    IF EXISTS (SELECT 1 FROM #W3_Special)
    BEGIN

        RAISERROR(
            N'2/6 Загрузка истории договоров клиентов промо',
            0, 1
        ) WITH NOWAIT;

        INSERT INTO #W3_History

        SELECT
              h.CLI_ID
            , h.CON_ID
            , MIN(CAST(h.DT_OPEN AS date))

        FROM LIQUIDITY.liq.DepositContract_all h
            WITH (NOLOCK)

        INNER JOIN
        (
            SELECT DISTINCT CLI_ID
            FROM #W3_Special
        ) sc
            ON sc.CLI_ID = h.CLI_ID

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

    END
    ELSE
    BEGIN

        RAISERROR(
            N'2/6 Специальных договоров нет. Загрузка истории пропущена.',
            0, 1
        ) WITH NOWAIT;

    END;

    CREATE UNIQUE CLUSTERED INDEX IX_History
        ON #W3_History(CLI_ID, CON_ID)
        WITH (MAXDOP = 1);

    --------------------------------------------------------
    -- Существующие клиенты и распределение по батчам
    --------------------------------------------------------

    SELECT
          CLI_ID

        , CAST(
            (ROW_NUMBER() OVER (ORDER BY CLI_ID) - 1)
            / @ClientBatchSize + 1
            AS int
          ) AS BatchNo

    INTO #W3_Prior

    FROM #W3_History

    WHERE dt_open < @DateFrom

    GROUP BY CLI_ID

    OPTION (MAXDOP 1, RECOMPILE);

    CREATE UNIQUE CLUSTERED INDEX IX_Prior
        ON #W3_Prior(CLI_ID)
        WITH (MAXDOP = 1);

    CREATE NONCLUSTERED INDEX IX_PriorBatch
        ON #W3_Prior(BatchNo, CLI_ID)
        WITH (MAXDOP = 1);

    SELECT
          @BatchCount = ISNULL(MAX(BatchNo), 0)
        , @Rows = COUNT(*)
    FROM #W3_Prior;

    SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

    RAISERROR(
        N'2/6 Готово: %d существующих клиентов, %d батчей, %d сек.',
        0, 1, @Rows, @BatchCount, @Sec
    ) WITH NOWAIT;

    --------------------------------------------------------
    -- 3. РАСЧЁТ НОВЫХ ДЕНЕГ ПО КЛИЕНТСКИМ БАТЧАМ
    --------------------------------------------------------

    -- Сохраняем все статусы, включая 0.
    -- Соединение по CON_ID строго как в v1.6.

    SELECT TOP (0)
          CON_ID
        , CAST(0 AS int) AS new_money_status

    INTO #W3_Allocation

    FROM #W3_Special;

    SET @BatchNo = 1;

    IF @BatchCount = 0
    BEGIN

        RAISERROR(
            N'3/6 Расчёт исторических остатков не требуется',
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
        -- Остаток специального вклада на дату открытия
        ----------------------------------------------------

        SELECT
              s.CON_ID
            , s.CLI_ID
            , s.dt_open

            , CAST(
                ISNULL(b.open_balance, 0)
                AS decimal(28,6)
              ) AS open_amount

        INTO #W3_Open

        FROM #W3_Special s

        INNER JOIN #W3_Prior p
            ON p.CLI_ID = s.CLI_ID
           AND p.BatchNo = @BatchNo

        OUTER APPLY
        (
            SELECT
                SUM(CAST(b.OUT_RUB AS decimal(28,6)))
                    AS open_balance

            FROM LIQUIDITY.liq.DepositContract_Saldo b
                WITH (NOLOCK)

            WHERE b.CON_ID = s.CON_ID

              AND b.DT_FROM < DATEADD(day, 1, s.dt_open)

              AND
              (
                  b.DT_TO IS NULL
                  OR b.DT_TO >= s.dt_open
              )

              AND b.OUT_RUB IS NOT NULL

        ) b

        OPTION (MAXDOP 1, RECOMPILE);

        CREATE UNIQUE CLUSTERED INDEX IX_Open
            ON #W3_Open(CLI_ID, dt_open, CON_ID)
            WITH (MAXDOP = 1);

        ----------------------------------------------------
        -- Суммарный объём открытий за день
        ----------------------------------------------------

        SELECT
              CLI_ID
            , dt_open
            , SUM(open_amount) AS total_open_amount

        INTO #W3_Days

        FROM #W3_Open

        GROUP BY CLI_ID, dt_open

        OPTION (MAXDOP 1, RECOMPILE);

        CREATE UNIQUE CLUSTERED INDEX IX_Days
            ON #W3_Days(CLI_ID, dt_open)
            WITH (MAXDOP = 1);

        ----------------------------------------------------
        -- Окна анализа остатков
        -- Историю до 01.10.2026 НЕ ограничиваем
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

        INTO #W3_Windows

        FROM #W3_Days

        GROUP BY
              CLI_ID
            , YEAR(dt_open)
            , MONTH(dt_open)

        OPTION (MAXDOP 1, RECOMPILE);

        ----------------------------------------------------
        -- Событийный расчёт остатков строго из v1.6
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

            FROM #W3_Windows w

            INNER JOIN #W3_History h
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
            FROM #W3_Windows

            UNION ALL

            SELECT
                  CLI_ID
                , month_start
                , month_start
                , CAST(0 AS decimal(28,6))
            FROM #W3_Windows

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
            FROM #W3_Days
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

            FROM #W3_Days d

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
        -- Исходные статусы v1.6:
        -- 0 = не новые деньги
        -- 1 = частично новые деньги
        -- 2 = полностью новые деньги
        ----------------------------------------------------

        INSERT INTO #W3_Allocation

        SELECT
              s.CON_ID

            , CASE
                WHEN s.open_amount <= 0
                  OR d.total_open_amount <= 0
                    THEN 0

                WHEN d.new_money_budget
                     >= d.total_open_amount - 0.01
                    THEN 2

                WHEN d.new_money_budget > 0
                    THEN 1

                ELSE 0
              END AS new_money_status

        FROM #W3_Open s

        INNER JOIN day_budget d
            ON d.CLI_ID = s.CLI_ID
           AND d.dt_open = s.dt_open

        OPTION (MAXDOP 1, RECOMPILE);

        SET @Rows = @@ROWCOUNT;

        DROP TABLE
            #W3_Open,
            #W3_Days,
            #W3_Windows;

        SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

        RAISERROR(
            N'3/6 Батч %d/%d завершён: %d строк, %d сек.',
            0, 1, @BatchNo, @BatchCount, @Rows, @Sec
        ) WITH NOWAIT;

        IF @BatchNo < @BatchCount
            WAITFOR DELAY @BatchPause;

        SET @BatchNo = @BatchNo + 1;

    END;

    CREATE NONCLUSTERED INDEX IX_Allocation
        ON #W3_Allocation(CON_ID)
        INCLUDE (new_money_status)
        WITH (MAXDOP = 1);

    DROP TABLE #W3_History, #W3_Special;

    --------------------------------------------------------
    -- 4. КЛАССИФИКАЦИЯ ВКЛАДОВ
    --------------------------------------------------------

    SET @StepTime = SYSDATETIME();

    RAISERROR(
        N'4/6 Классификация вкладов и агрегация по бакетам',
        0, 1
    ) WITH NOWAIT;

    ;WITH joined AS
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

            , CASE
                WHEN s.special_2m = 1
                 AND s.CLI_ID IS NOT NULL
                 AND p.CLI_ID IS NULL
                    THEN 1
                ELSE 0
              END AS is_new_client

            , ISNULL(sa.new_money_status, 0)
                AS special_new_money_status

        FROM #W3_Src s

        LEFT JOIN #W3_Prior p
            ON p.CLI_ID = s.CLI_ID

        -- Ключ соединения строго из v1.6
        LEFT JOIN #W3_Allocation sa
            ON sa.CON_ID = s.CON_ID

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
                      WHEN ISNULL(s.conv, 'AT_THE_END')
                           = 'AT_THE_END'
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
                WHEN ISNULL(TRY_CAST(j.[Нов] AS int), 0) = 1
                    THEN 1
                ELSE 0
              END AS is_nov

            , CASE
                WHEN ISNULL(TRY_CAST(j.[НДП] AS int), 0) = 1
                  OR ISNULL(TRY_CAST(j.[НДМ] AS int), 0) = 1
                    THEN 1
                ELSE 0
              END AS is_ndp_ndm

            , CASE
                WHEN ISNULL(TRY_CAST(j.[Пнс] AS int), 0) = 1
                  OR ISNULL(TRY_CAST(j.[Мпл] AS int), 0) = 1
                    THEN 1
                ELSE 0
              END AS is_pns_mpl

            , CASE
                WHEN ISNULL(TRY_CAST(j.[От1] AS int), 0) = 1
                  OR ISNULL(TRY_CAST(j.[Пк2] AS int), 0) = 1
                    THEN 1
                ELSE 0
              END AS retention

        FROM joined j
    ),

    classified AS
    (
        SELECT
              m.OUT_RUB
            , m.rate_con

            , CASE

                WHEN m.is_mp = 1
                    THEN N'Маркетплейсы'

                WHEN m.PROD_NAME_res = N'Активный процент'
                    THEN N'Активный процент'

                ------------------------------------------------
                -- Специальное промо 15% с 01.10.2026
                ------------------------------------------------

                WHEN m.special_2m = 1
                 AND m.is_new_client = 1
                    THEN N'Новые клиенты'

                WHEN m.special_2m = 1
                 AND m.special_new_money_status = 2
                    THEN N'Новые деньги'

                WHEN m.special_2m = 1
                 AND m.special_new_money_status = 1
                    THEN N'Частично новые деньги'

                WHEN m.special_2m = 1
                    THEN N'Другие вклады со ставкой новых денег'

                ------------------------------------------------
                -- Обычная классификация
                -- Включая вклады 15%, открытые до октября
                ------------------------------------------------

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
    )

    --------------------------------------------------------
    -- Материализация агрегации по исходным сегментам
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

        , SUM(
            CAST(OUT_RUB AS decimal(24,2))
            * CAST(rate_con AS decimal(12,8))
          ) AS weighted_rate_sum

    INTO #W3_Agg

    FROM classified

    GROUP BY GROUPING SETS
    (
          (bucket_code, segment_name)
        , (bucket_code)
    )

    OPTION (MAXDOP 1, RECOMPILE);

    DROP TABLE
        #W3_Src,
        #W3_Prior,
        #W3_Allocation;

    --------------------------------------------------------
    -- Воспроизводим выходные поля v1.6
    -- С исходными округлениями
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
            / NULLIF(a.segment_vol, 0)
            AS decimal(9,6)
          ) AS ClientRate

    INTO #W3_Raw

    FROM #W3_Agg a

    CROSS JOIN
    (
        SELECT SUM(segment_vol) AS grand_total
        FROM #W3_Agg
        WHERE is_total = 0
    ) t;

    DROP TABLE #W3_Agg;

    SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

    RAISERROR(
        N'4/6 Классификация завершена: %d сек.',
        0, 1, @Sec
    ) WITH NOWAIT;

    SET @Sec = DATEDIFF(second, @WeekTime, SYSDATETIME());

    RAISERROR(
        N'Оптимизированный расчёт недели: %d сек.',
        0, 1, @Sec
    ) WITH NOWAIT;

    --------------------------------------------------------
    -- 5. КОНТРОЛЬНАЯ СВЕРКА С ИСХОДНОЙ v1.6
    --
    -- Сверяем только первую неделю и только если она
    -- полностью находится после начала специального промо.
    --
    -- До 01.10.2026 логика намеренно изменена.
    --------------------------------------------------------

    IF @VerifyFirstWeek = 1
       AND @WeekNo = 1
       AND @DateFrom >= @Promo15From
    BEGIN

        SET @StepTime = SYSDATETIME();

        RAISERROR(
            N'5/6 Контрольный запуск исходной v1.6',
            0, 1
        ) WITH NOWAIT;

        SELECT
              Bucket
            , Segment
            , SegmentVolume
            , SegmentSharePct
            , ClientRate

        INTO #W3_Reference

        FROM ALM_TEST.reports.
            [fn_NewAttractionVolumes_forDimaMASS_v1.6]
        (
              @DateTo
            , @DateFrom
            , @DateTo
            , 0
        )

        OPTION (MAXDOP 1, RECOMPILE);

        IF EXISTS
        (
            SELECT * FROM #W3_Raw

            EXCEPT

            SELECT * FROM #W3_Reference
        )
        OR EXISTS
        (
            SELECT * FROM #W3_Reference

            EXCEPT

            SELECT * FROM #W3_Raw
        )
        BEGIN

            SELECT
                  N'Оптимизированный' AS SourceVersion
                , d.*

            FROM
            (
                SELECT * FROM #W3_Raw

                EXCEPT

                SELECT * FROM #W3_Reference
            ) d

            UNION ALL

            SELECT
                  N'Исходная v1.6'
                , d.*

            FROM
            (
                SELECT * FROM #W3_Reference

                EXCEPT

                SELECT * FROM #W3_Raw
            ) d

            ORDER BY Bucket, Segment, SourceVersion;

            THROW 50002,
                N'Обнаружено расхождение с v1.6. Расчёт остановлен.',
                1;

        END;

        DROP TABLE #W3_Reference;

        SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

        RAISERROR(
            N'5/6 ПРОВЕРКА ПРОЙДЕНА. Полное совпадение. v1.6: %d сек.',
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
    -- 6. ВОСПРОИЗВОДИМ ИСХОДНЫЙ СКРИПТ КОЛЛЕГ
    --------------------------------------------------------

    SET @StepTime = SYSDATETIME();

    ;WITH mapped AS
    (
        SELECT
            Bucket,

            CASE
                WHEN Segment = N'Остальные вклады'
                    THEN N'Базовая ставка'

                WHEN Segment IN
                (
                    N'Автоматический механизм удержания',
                    N'Другие вклады со ставкой новых денег',
                    N'Новые деньги без справочника'
                )
                    THEN N'Удержание'

                WHEN Segment IN
                (
                    N'Частично новые деньги',
                    N'Новые деньги'
                )
                    THEN N'Новые деньги'

                WHEN Segment = N'Вклады ПНС и МПЛ'
                    THEN N'Пенсионеры и Маркетплейсы'

                ELSE Segment

            END AS Segment,

            SegmentVolume,
            SegmentSharePct,
            ClientRate

        FROM #W3_Raw
    )

    --------------------------------------------------------
    -- Исходная выгрузка коллег:
    -- бакет + укрупнённая группа вкладов
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

    INTO #W3_Mapped

    FROM mapped

    GROUP BY Bucket, Segment

    OPTION (MAXDOP 1, RECOMPILE);

    --------------------------------------------------------
    -- ФИНАЛЬНАЯ АГРЕГАЦИЯ
    -- Аналог расчёта коллег в Excel
    --------------------------------------------------------

    INSERT INTO #W3_Result
    SELECT
          @DateFrom
        , @DateTo
        , Segment

        ----------------------------------------------------
        -- А. Общий объём, млрд руб.
        ----------------------------------------------------

        , CAST(
            SUM(SegmentVolume)
            AS decimal(18,6)
          )

        ----------------------------------------------------
        -- Б. Сумма долей от общего объёма
        ----------------------------------------------------

        , CAST(
            SUM(SegmentSharePct)
            AS decimal(18,4)
          )

        ----------------------------------------------------
        -- В. Средневзвешенная клиентская ставка
        ----------------------------------------------------

        , CAST(
            CAST(
                SUM(SegmentVolume * ClientRate)
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

    FROM #W3_Mapped

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
        #W3_Raw,
        #W3_Mapped;

    SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

    RAISERROR(
        N'6/6 Итоговая агрегация: %d строк, %d сек.',
        0, 1, @Rows, @Sec
    ) WITH NOWAIT;

    SET @Sec = DATEDIFF(second, @WeekTime, SYSDATETIME());

    RAISERROR(
        N'===== НЕДЕЛЯ %d/%d ЗАВЕРШЕНА: %d сек. =====',
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
    N'РАСЧЁТ ЗАВЕРШЁН. Общее время: %d сек.',
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

FROM #W3_Result

ORDER BY
    WeekEnd DESC,

    CASE Segment
        WHEN N'Базовая ставка' THEN 1
        WHEN N'Удержание' THEN 2
        WHEN N'Новые деньги' THEN 3
        WHEN N'Новые клиенты' THEN 4
        WHEN N'Активный процент' THEN 5
        WHEN N'Маркетплейсы' THEN 6
        WHEN N'Пенсионеры и Маркетплейсы' THEN 7
        WHEN N'Итого' THEN 99
        ELSE 98
    END;




    DECLARE @AsOfDate date = '20261125';
DECLARE @Weeks int = 12;
DECLARE @Promo15From date = '20261001';
DECLARE @VerifyFirstWeek bit = 1;
