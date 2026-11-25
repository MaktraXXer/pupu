USE ALM_TEST;
SET NOCOUNT ON;

------------------------------------------------------------
-- НАСТРОЙКИ
------------------------------------------------------------

DECLARE @AsOfDate date = '20261125'; -- Или CAST(GETDATE() AS date)
DECLARE @Weeks int = 12;
DECLARE @DataLag int = 2;

DECLARE @ClientBatchSize int = 3000;

DECLARE @BatchPause varchar(8) = '00:00:01';
DECLARE @WeekPause  varchar(8) = '00:00:03';

IF @Weeks NOT BETWEEN 1 AND 12 OR @ClientBatchSize <= 0
    THROW 50001, N'Некорректные параметры расчёта', 1;

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
DECLARE @HistoryRows int;

DECLARE @FromText varchar(10);
DECLARE @ToText varchar(10);

DECLARE @StartTime datetime2 = SYSDATETIME();
DECLARE @WeekTime datetime2;
DECLARE @StepTime datetime2;

------------------------------------------------------------
-- ОЧИСТКА ПОСЛЕ ПРЕДЫДУЩИХ ЗАПУСКОВ
------------------------------------------------------------

DROP TABLE IF EXISTS
    #WA_Result,
    #WA_Src,
    #WA_Special,
    #WA_History,
    #WA_Prior,
    #WA_Open,
    #WA_Days,
    #WA_NewMoney;

CREATE TABLE #WA_Result
(
    WeekStart    date,
    WeekEnd      date,
    Segment      nvarchar(100),
    Volume       decimal(18,6),
    SharePct     decimal(18,6),
    ClientRate   decimal(9,6),
    WeightedTerm decimal(12,2)
);

------------------------------------------------------------
-- ОСНОВНОЙ ЦИКЛ: ОДНА НЕДЕЛЯ = ОДИН БАТЧ
------------------------------------------------------------

WHILE @WeekNo <= @Weeks
BEGIN

    SET @WeekTime = SYSDATETIME();
    SET @DateFrom = DATEADD(day, -6, @DateTo);

    SET @FromText = CONVERT(varchar(10), @DateFrom, 104);
    SET @ToText   = CONVERT(varchar(10), @DateTo, 104);

    RAISERROR(
        N'===== НЕДЕЛЯ %d/%d: %s - %s =====',
        0, 1,
        @WeekNo, @Weeks, @FromText, @ToText
    ) WITH NOWAIT;

    --------------------------------------------------------
    -- ШАГ 1. ОДНОКРАТНОЕ ЧТЕНИЕ БАЛАНСОВОГО СРЕЗА
    --------------------------------------------------------

    SET @StepTime = SYSDATETIME();

    SELECT
        v.CON_ID,
        v.CLI_ID,
        CAST(v.dt_open AS date) AS DtOpen,
        v.OUT_RUB AS OutRub,
        v.rate_con AS RateCon,
        v.termdays AS TermDays,
        c.ConvType,

        CASE
            WHEN v.PROD_NAME_res IN
            (
                N'ДОМа надёжно',
                N'Всё в ДОМ',
                N'Надёжный',
                N'Надёжный VIP',
                N'Надёжный премиум',
                N'Надёжныйпромо',
                N'Надёжный промо',
                N'Надёжный старт',
                N'Надёжный T2',
                N'Надёжный Т2',
                N'Надёжный Мегафон',
                N'Надёжныйпрайм',
                N'Надёжный прайм',
                N'Надёжный процент',
                N'Надёжныйпроцент',
                N'Могучий'
            ) THEN 6

            WHEN v.PROD_NAME_res = N'Активный процент'
                THEN 5

            WHEN v.termdays BETWEEN 45 AND 70
             AND c.ConvType = 'AT_THE_END'
             AND ABS(v.rate_con - 0.1500) <= 0.0005
                THEN 0

            ELSE 8
        END AS BaseCode,

        CASE
            WHEN v.termdays BETWEEN 26 AND 44 THEN 31
            WHEN v.termdays BETWEEN 45 AND 70 THEN 61
            WHEN v.termdays BETWEEN 85 AND 119 THEN 91
            WHEN v.termdays BETWEEN 120 AND 140 THEN 124
            WHEN v.termdays BETWEEN 141 AND 170 THEN 151
            WHEN v.termdays BETWEEN 175 AND 200 THEN 181
            WHEN v.termdays BETWEEN 201 AND 230 THEN 212
            WHEN v.termdays BETWEEN 231 AND 250 THEN 243
            WHEN v.termdays BETWEEN 251 AND 290 THEN 274
            WHEN v.termdays BETWEEN 340 AND 405 THEN 365
            WHEN v.termdays BETWEEN 540 AND 621 THEN 550
            WHEN v.termdays BETWEEN 720 AND 763 THEN 750
            WHEN v.termdays BETWEEN 1090 AND 1140 THEN 1100
            WHEN v.termdays BETWEEN 1450 AND 1475 THEN 1460
            WHEN v.termdays BETWEEN 1795 AND 1830 THEN 1825
            ELSE v.termdays
        END AS Bucket

    INTO #WA_Src

    FROM ALM.ALM.vw_balance_rest_all v WITH (NOLOCK)

    CROSS APPLY
    (
        VALUES
        (
            CASE
                WHEN ISNULL(v.conv, 'AT_THE_END') = 'AT_THE_END'
                    THEN 'AT_THE_END'
                ELSE 'NOT_AT_THE_END'
            END
        )
    ) c(ConvType)

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
          FROM LIQUIDITY.liq.man_FloatContracts fc WITH (NOLOCK)
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
        N'1/4 Балансовый срез: %d строк. Время: %d сек.',
        0, 1, @Rows, @Sec
    ) WITH NOWAIT;

    IF @Rows = 0
    BEGIN
        RAISERROR(
            N'ВНИМАНИЕ: нет данных за неделю. Неделя пропущена.',
            0, 1
        ) WITH NOWAIT;
    END
    ELSE
    BEGIN

        ----------------------------------------------------
        -- ШАГ 2. ИСТОРИЯ ТОЛЬКО КЛИЕНТОВ ПРОМО 15%
        ----------------------------------------------------

        SET @StepTime = SYSDATETIME();

        SELECT DISTINCT
            CLI_ID,
            CON_ID,
            DtOpen
        INTO #WA_Special
        FROM #WA_Src
        WHERE BaseCode = 0
          AND CLI_ID IS NOT NULL
          AND CON_ID IS NOT NULL
        OPTION (MAXDOP 1, RECOMPILE);

        CREATE UNIQUE CLUSTERED INDEX IX_Special
            ON #WA_Special(CLI_ID, DtOpen, CON_ID)
            WITH (MAXDOP = 1);

        -- Наследуем типы идентификаторов из источника
        SELECT TOP (0)
            h.CLI_ID,
            h.CON_ID,
            CAST(h.DT_OPEN AS date) AS DtOpen
        INTO #WA_History
        FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK);

        SET @HistoryRows = 0;

        IF EXISTS (SELECT 1 FROM #WA_Special)
        BEGIN

            INSERT INTO #WA_History
            SELECT
                h.CLI_ID,
                h.CON_ID,
                MIN(CAST(h.DT_OPEN AS date))

            FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK)

            INNER JOIN
            (
                SELECT DISTINCT CLI_ID
                FROM #WA_Special
            ) s
                ON s.CLI_ID = h.CLI_ID

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

            SET @HistoryRows = @@ROWCOUNT;

        END;

        CREATE UNIQUE CLUSTERED INDEX IX_History
            ON #WA_History(CLI_ID, CON_ID)
            WITH (MAXDOP = 1);

        -- Существующие клиенты и номера их батчей
        SELECT
            CLI_ID,

            CAST(
                (ROW_NUMBER() OVER (ORDER BY CLI_ID) - 1)
                / @ClientBatchSize + 1
                AS int
            ) AS BatchNo

        INTO #WA_Prior

        FROM #WA_History

        WHERE DtOpen < @DateFrom

        GROUP BY CLI_ID

        OPTION (MAXDOP 1, RECOMPILE);

        CREATE UNIQUE CLUSTERED INDEX IX_PriorBatch
            ON #WA_Prior(BatchNo, CLI_ID)
            WITH (MAXDOP = 1);

        CREATE UNIQUE NONCLUSTERED INDEX IX_PriorClient
            ON #WA_Prior(CLI_ID)
            WITH (MAXDOP = 1);

        SELECT
            @BatchCount = ISNULL(MAX(BatchNo), 0),
            @Rows = COUNT(*)
        FROM #WA_Prior;

        SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

        RAISERROR(
            N'2/4 История: %d договоров, %d существующих клиентов, %d батчей. Время: %d сек.',
            0, 1,
            @HistoryRows, @Rows, @BatchCount, @Sec
        ) WITH NOWAIT;

        ----------------------------------------------------
        -- ШАГ 3. НОВЫЕ ДЕНЬГИ БАТЧАМИ ПО КЛИЕНТАМ
        ----------------------------------------------------

        SELECT TOP (0)
            CLI_ID,
            CON_ID,
            DtOpen
        INTO #WA_NewMoney
        FROM #WA_Special;

        CREATE UNIQUE CLUSTERED INDEX IX_NewMoney
            ON #WA_NewMoney(CLI_ID, DtOpen, CON_ID)
            WITH (MAXDOP = 1);

        SET @BatchNo = 1;

        WHILE @BatchNo <= @BatchCount
        BEGIN

            SET @StepTime = SYSDATETIME();

            RAISERROR(
                N'3/4 Остатки: начало батча %d/%d',
                0, 1, @BatchNo, @BatchCount
            ) WITH NOWAIT;

            ------------------------------------------------
            -- Остатки специальных вкладов на дату открытия
            ------------------------------------------------

            SELECT
                s.CLI_ID,
                s.CON_ID,
                s.DtOpen,

                CAST(
                    ISNULL(b.OpenBalance, 0)
                    AS decimal(28,6)
                ) AS OpenAmount

            INTO #WA_Open

            FROM #WA_Special s

            INNER JOIN #WA_Prior p
                ON p.CLI_ID = s.CLI_ID
               AND p.BatchNo = @BatchNo

            OUTER APPLY
            (
                SELECT
                    SUM(CAST(b.OUT_RUB AS decimal(28,6)))
                        AS OpenBalance

                FROM LIQUIDITY.liq.DepositContract_Saldo b
                    WITH (NOLOCK)

                WHERE b.CON_ID = s.CON_ID
                  AND b.DT_FROM < DATEADD(day, 1, s.DtOpen)

                  AND
                  (
                      b.DT_TO IS NULL
                      OR b.DT_TO >= s.DtOpen
                  )

                  AND b.OUT_RUB IS NOT NULL
            ) b

            OPTION (MAXDOP 1, RECOMPILE);

            CREATE UNIQUE CLUSTERED INDEX IX_Open
                ON #WA_Open(CLI_ID, DtOpen, CON_ID)
                WITH (MAXDOP = 1);

            SELECT
                CLI_ID,
                DtOpen,
                SUM(OpenAmount) AS TotalOpenAmount

            INTO #WA_Days

            FROM #WA_Open

            GROUP BY CLI_ID, DtOpen

            OPTION (MAXDOP 1, RECOMPILE);

            CREATE UNIQUE CLUSTERED INDEX IX_Days
                ON #WA_Days(CLI_ID, DtOpen)
                WITH (MAXDOP = 1);

            ------------------------------------------------
            -- Исторические остатки только текущего батча
            ------------------------------------------------

            ;WITH analysis_months AS
            (
                SELECT
                    CLI_ID,
                    DATEFROMPARTS(
                        YEAR(DtOpen), MONTH(DtOpen), 1
                    ) AS MonthStart,
                    MAX(DtOpen) AS MaxOpen

                FROM #WA_Days

                GROUP BY CLI_ID, YEAR(DtOpen), MONTH(DtOpen)
            ),

            analysis_windows AS
            (
                SELECT
                    CLI_ID,
                    MonthStart,
                    DATEADD(month, -1, MonthStart) AS PrevStart,
                    MaxOpen
                FROM analysis_months
            ),

            saldo_clipped AS
            (
                SELECT
                    w.CLI_ID,
                    w.MonthStart,
                    w.MaxOpen,

                    CAST(s.OUT_RUB AS decimal(28,6)) AS OutRub,

                    bounds.EffectiveFrom,
                    bounds.EffectiveTo

                FROM analysis_windows w

                INNER JOIN #WA_History h
                    ON h.CLI_ID = w.CLI_ID
                   AND h.DtOpen <= w.MaxOpen

                INNER JOIN LIQUIDITY.liq.DepositContract_Saldo s
                    WITH (NOLOCK)

                    ON s.CON_ID = h.CON_ID
                   AND s.DT_FROM < DATEADD(day, 1, w.MaxOpen)

                   AND
                   (
                       s.DT_TO IS NULL
                       OR s.DT_TO >= w.PrevStart
                   )

                CROSS APPLY
                (
                    SELECT
                        (
                            SELECT MAX(v.dt)
                            FROM
                            (
                                VALUES
                                    (CAST(s.DT_FROM AS date)),
                                    (h.DtOpen),
                                    (w.PrevStart)
                            ) v(dt)
                        ) AS EffectiveFrom,

                        (
                            SELECT MIN(v.dt)
                            FROM
                            (
                                VALUES
                                (
                                    ISNULL(
                                        CAST(s.DT_TO AS date),
                                        w.MaxOpen
                                    )
                                ),
                                (w.MaxOpen)
                            ) v(dt)
                        ) AS EffectiveTo
                ) bounds

                WHERE s.OUT_RUB IS NOT NULL
                  AND bounds.EffectiveFrom <= bounds.EffectiveTo
            ),

            saldo_events AS
            (
                SELECT
                    s.CLI_ID,
                    s.MonthStart,
                    e.EventDate,
                    e.Delta

                FROM saldo_clipped s

                CROSS APPLY
                (
                    VALUES
                    (
                        s.EffectiveFrom,
                        s.OutRub
                    ),
                    (
                        CASE
                            WHEN s.EffectiveTo < s.MaxOpen
                            THEN DATEADD(day, 1, s.EffectiveTo)
                        END,
                        -s.OutRub
                    )
                ) e(EventDate, Delta)

                WHERE e.EventDate IS NOT NULL

                UNION ALL

                SELECT
                    CLI_ID,
                    MonthStart,
                    PrevStart,
                    CAST(0 AS decimal(28,6))
                FROM analysis_windows

                UNION ALL

                SELECT
                    CLI_ID,
                    MonthStart,
                    MonthStart,
                    CAST(0 AS decimal(28,6))
                FROM analysis_windows

                UNION ALL

                SELECT
                    CLI_ID,
                    DATEFROMPARTS(YEAR(DtOpen), MONTH(DtOpen), 1),
                    DtOpen,
                    CAST(0 AS decimal(28,6))
                FROM #WA_Days
            ),

            event_agg AS
            (
                SELECT
                    CLI_ID,
                    MonthStart,
                    EventDate,
                    SUM(Delta) AS Delta

                FROM saldo_events

                GROUP BY CLI_ID, MonthStart, EventDate
            ),

            timeline AS
            (
                SELECT
                    CLI_ID,
                    MonthStart,
                    EventDate,

                    SUM(Delta) OVER
                    (
                        PARTITION BY CLI_ID, MonthStart
                        ORDER BY EventDate
                        ROWS BETWEEN UNBOUNDED PRECEDING
                                 AND CURRENT ROW
                    ) AS Balance

                FROM event_agg
            ),

            timeline_stats AS
            (
                SELECT
                    CLI_ID,
                    MonthStart,
                    EventDate,
                    Balance,

                    MAX
                    (
                        CASE
                            WHEN EventDate < MonthStart
                            THEN Balance
                        END
                    ) OVER
                    (
                        PARTITION BY CLI_ID, MonthStart
                    ) AS PrevMonthPeak,

                    MAX
                    (
                        CASE
                            WHEN EventDate >= MonthStart
                            THEN Balance
                        END
                    ) OVER
                    (
                        PARTITION BY CLI_ID, MonthStart
                        ORDER BY EventDate
                        ROWS BETWEEN UNBOUNDED PRECEDING
                                 AND 1 PRECEDING
                    ) AS CurrentMonthPeakBefore

                FROM timeline
            ),

            day_balances AS
            (
                SELECT
                    d.CLI_ID,
                    d.DtOpen,
                    d.TotalOpenAmount,
                    ISNULL(t.Balance, 0) AS BalanceToday,

                    CASE
                        WHEN ISNULL(t.PrevMonthPeak, 0)
                          >= ISNULL(t.CurrentMonthPeakBefore, 0)
                            THEN ISNULL(t.PrevMonthPeak, 0)

                        ELSE ISNULL(t.CurrentMonthPeakBefore, 0)
                    END AS ComparisonPeak

                FROM #WA_Days d

                LEFT JOIN timeline_stats t
                    ON t.CLI_ID = d.CLI_ID

                   AND t.MonthStart =
                       DATEFROMPARTS(
                           YEAR(d.DtOpen), MONTH(d.DtOpen), 1
                       )

                   AND t.EventDate = d.DtOpen
            ),

            day_budget AS
            (
                SELECT
                    CLI_ID,
                    DtOpen,
                    TotalOpenAmount,

                    CASE
                        WHEN TotalOpenAmount <= 0 THEN 0

                        WHEN BalanceToday <= ComparisonPeak
                            THEN 0

                        WHEN BalanceToday - ComparisonPeak
                             >= TotalOpenAmount
                            THEN TotalOpenAmount

                        ELSE BalanceToday - ComparisonPeak
                    END AS NewMoneyBudget

                FROM day_balances
            )

            -- Полностью и частично новые деньги объединены

            INSERT INTO #WA_NewMoney
            SELECT
                s.CLI_ID,
                s.CON_ID,
                s.DtOpen

            FROM #WA_Open s

            INNER JOIN day_budget d
                ON d.CLI_ID = s.CLI_ID
               AND d.DtOpen = s.DtOpen

            WHERE s.OpenAmount > 0
              AND d.TotalOpenAmount > 0

              AND
              (
                  d.NewMoneyBudget >= d.TotalOpenAmount - 0.01
                  OR d.NewMoneyBudget > 0
              )

            OPTION (MAXDOP 1, RECOMPILE);

            SET @Rows = @@ROWCOUNT;

            DROP TABLE #WA_Open, #WA_Days;

            SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

            RAISERROR(
                N'3/4 Батч %d/%d завершён. Новые деньги: %d договоров. Время: %d сек.',
                0, 1,
                @BatchNo, @BatchCount, @Rows, @Sec
            ) WITH NOWAIT;

            IF @BatchNo < @BatchCount
                WAITFOR DELAY @BatchPause;

            SET @BatchNo = @BatchNo + 1;

        END;

        ----------------------------------------------------
        -- ШАГ 4. КЛАССИФИКАЦИЯ И АГРЕГАЦИЯ ВСЕЙ НЕДЕЛИ
        ----------------------------------------------------

        SET @StepTime = SYSDATETIME();

        RAISERROR(
            N'4/4 Начало итоговой классификации',
            0, 1
        ) WITH NOWAIT;

        ;WITH classified AS
        (
            SELECT
                s.Bucket,

                CAST(s.OutRub AS decimal(24,2)) AS VolumeRub,

                CAST(s.OutRub AS decimal(24,2))
                    * CAST(s.RateCon AS decimal(12,8))
                    AS WeightedRate,

                CASE
                    -- Маркетплейсы и Активный процент
                    WHEN s.BaseCode IN (5, 6)
                        THEN s.BaseCode

                    -- Специальное промо 15%
                    WHEN s.BaseCode = 0
                        THEN
                            CASE
                                WHEN s.CLI_ID IS NOT NULL
                                 AND p.CLI_ID IS NULL
                                    THEN 4

                                WHEN nm.CON_ID IS NOT NULL
                                    THEN 3

                                ELSE 2
                            END

                    -- Обычная классификация
                    WHEN f.Retention = 1 THEN 2

                    WHEN f.IsPnsMpl = 1 THEN 7

                    WHEN f.IsNov = 1
                     AND d.DictMatch = 1 THEN 4

                    WHEN f.IsNdpNdm = 1
                     AND d.DictMatch = 1 THEN 3

                    WHEN f.IsNov = 1
                      OR f.IsNdpNdm = 1
                      OR d.DictMatch = 1 THEN 2

                    ELSE 1
                END AS SegmentCode

            FROM #WA_Src s

            LEFT JOIN #WA_Prior p
                ON s.BaseCode = 0
               AND p.CLI_ID = s.CLI_ID

            LEFT JOIN #WA_NewMoney nm
                ON s.BaseCode = 0
               AND nm.CLI_ID = s.CLI_ID
               AND nm.CON_ID = s.CON_ID
               AND nm.DtOpen = s.DtOpen

            -- Надбавки проверяются только у обычных вкладов

            OUTER APPLY
            (
                SELECT TOP (1)
                    a.[Нов],
                    a.[НДП],
                    a.[НДМ],
                    a.[Мпл],
                    a.[Пнс],
                    a.[От1],
                    a.[Пк2]

                FROM ALM.ehd.attr_DepoFLConditions a
                    WITH (NOLOCK)

                WHERE a.CON_ID = s.CON_ID
                  AND s.BaseCode = 8

                ORDER BY
                    a.DT_UPDATE DESC,
                    a.loaddate DESC
            ) a

            CROSS APPLY
            (
                SELECT
                    CASE
                        WHEN TRY_CAST(a.[Нов] AS int) = 1
                        THEN 1 ELSE 0
                    END AS IsNov,

                    CASE
                        WHEN TRY_CAST(a.[НДП] AS int) = 1
                          OR TRY_CAST(a.[НДМ] AS int) = 1
                        THEN 1 ELSE 0
                    END AS IsNdpNdm,

                    CASE
                        WHEN TRY_CAST(a.[Пнс] AS int) = 1
                          OR TRY_CAST(a.[Мпл] AS int) = 1
                        THEN 1 ELSE 0
                    END AS IsPnsMpl,

                    CASE
                        WHEN TRY_CAST(a.[От1] AS int) = 1
                          OR TRY_CAST(a.[Пк2] AS int) = 1
                        THEN 1 ELSE 0
                    END AS Retention
            ) f

            -- Справочник проверяется только там,
            -- где он влияет на итоговую классификацию

            OUTER APPLY
            (
                SELECT TOP (1)
                    1 AS DictMatch

                FROM ALM_TEST.WORK.promo_new_money_rate_dict d
                    WITH (NOLOCK)

                WHERE s.BaseCode = 8

                  AND f.Retention = 0
                  AND f.IsPnsMpl = 0

                  AND s.DtOpen BETWEEN d.date_from AND d.date_to

                  AND s.TermDays BETWEEN d.term_min AND d.term_max

                  AND s.ConvType = d.conv_type

                  AND s.OutRub BETWEEN d.amount_from AND d.amount_to

                  AND ABS(s.RateCon - d.promo_rate) <= 0.0005
            ) d
        ),

        agg AS
        (
            SELECT
                SegmentCode,

                GROUPING(SegmentCode) AS IsTotal,

                SUM(VolumeRub) AS VolumeRub,

                SUM(WeightedRate) AS WeightedRateSum,

                SUM(
                    VolumeRub * CAST(Bucket AS decimal(12,4))
                ) AS WeightedTermSum

            FROM classified

            GROUP BY GROUPING SETS
            (
                (SegmentCode),
                ()
            )

            HAVING COUNT_BIG(*) > 0
        ),

        totals AS
        (
            SELECT
                *,

                MAX(
                    CASE
                        WHEN IsTotal = 1 THEN VolumeRub
                    END
                ) OVER () AS TotalVolumeRub

            FROM agg
        )

        INSERT INTO #WA_Result
        SELECT
            @DateFrom,
            @DateTo,

            CASE
                WHEN IsTotal = 1 THEN N'Итого'
                WHEN SegmentCode = 1 THEN N'Базовая ставка'
                WHEN SegmentCode = 2 THEN N'Механизм удержания'
                WHEN SegmentCode = 3 THEN N'Новые деньги'
                WHEN SegmentCode = 4 THEN N'Новые клиенты'
                WHEN SegmentCode = 5 THEN N'Активный процент'
                WHEN SegmentCode = 6 THEN N'Маркетплейсы'
                WHEN SegmentCode = 7 THEN N'Пенсионеры и Маркетплейсы'
            END,

            -- Объём, млрд руб.
            CAST(
                CAST(VolumeRub AS decimal(28,2))
                / 1000000000.0
                AS decimal(18,6)
            ),

            -- Доля от общего объёма недели
            CAST(
                CAST(VolumeRub AS decimal(28,2))
                / NULLIF(
                    CAST(TotalVolumeRub AS decimal(28,2)), 0
                )
                AS decimal(18,6)
            ),

            -- Средневзвешенная клиентская ставка
            CAST(
                CAST(WeightedRateSum AS decimal(30,10))
                / NULLIF(
                    CAST(VolumeRub AS decimal(28,2)), 0
                )
                AS decimal(9,6)
            ),

            -- Средневзвешенная срочность по бакетам
            CAST(
                CAST(WeightedTermSum AS decimal(30,6))
                / NULLIF(
                    CAST(VolumeRub AS decimal(28,2)), 0
                )
                AS decimal(12,2)
            )

        FROM totals

        OPTION (MAXDOP 1, RECOMPILE);

        SET @Rows = @@ROWCOUNT;
        SET @Sec = DATEDIFF(second, @StepTime, SYSDATETIME());

        RAISERROR(
            N'4/4 Агрегация завершена. Строк: %d. Время: %d сек.',
            0, 1, @Rows, @Sec
        ) WITH NOWAIT;

    END;

    --------------------------------------------------------
    -- ОСВОБОЖДАЕМ ПРОМЕЖУТОЧНЫЕ ТАБЛИЦЫ
    --------------------------------------------------------

    DROP TABLE IF EXISTS
        #WA_Src,
        #WA_Special,
        #WA_History,
        #WA_Prior,
        #WA_Open,
        #WA_Days,
        #WA_NewMoney;

    SET @Sec = DATEDIFF(second, @WeekTime, SYSDATETIME());

    RAISERROR(
        N'===== НЕДЕЛЯ %d/%d ЗАВЕРШЕНА. Время: %d сек. =====',
        0, 1,
        @WeekNo, @Weeks, @Sec
    ) WITH NOWAIT;

    IF @WeekNo < @Weeks
        WAITFOR DELAY @WeekPause;

    SET @DateTo = DATEADD(day, -7, @DateTo);
    SET @WeekNo = @WeekNo + 1;

END;

------------------------------------------------------------
-- ИТОГОВЫЙ РЕЗУЛЬТАТ
------------------------------------------------------------

SET @Sec = DATEDIFF(second, @StartTime, SYSDATETIME());

RAISERROR(
    N'РАСЧЁТ ЗАВЕРШЁН. Общее время: %d сек.',
    0, 1, @Sec
) WITH NOWAIT;

SELECT
    WeekStart,
    WeekEnd,
    Segment,
    Volume,
    SharePct,
    ClientRate,
    WeightedTerm

FROM #WA_Result

ORDER BY
    WeekEnd DESC,

    CASE Segment
        WHEN N'Базовая ставка' THEN 1
        WHEN N'Механизм удержания' THEN 2
        WHEN N'Новые деньги' THEN 3
        WHEN N'Новые клиенты' THEN 4
        WHEN N'Активный процент' THEN 5
        WHEN N'Маркетплейсы' THEN 6
        WHEN N'Пенсионеры и Маркетплейсы' THEN 7
        WHEN N'Итого' THEN 99
        ELSE 98
    END;
