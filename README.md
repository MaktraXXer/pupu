USE ALM_TEST;
GO

CREATE OR ALTER FUNCTION
    [reports].[fn_NewAttractionVolumes_forDimaMASS_Weekly_v2]
(
    @AsOfDate date,
    @Weeks int
)
RETURNS @Result TABLE
(
    WeekStart       date,
    WeekEnd         date,
    Segment         nvarchar(100),
    Volume          decimal(18,6),
    SharePct        decimal(18,6),
    ClientRate      decimal(9,6),
    WeightedTerm    decimal(12,2)
)
AS
BEGIN

    IF @Weeks IS NULL OR @Weeks NOT BETWEEN 1 AND 12
        RETURN;

    DECLARE @AvailableDate date =
        DATEADD(day, -2,
            ISNULL(@AsOfDate, CAST(GETDATE() AS date)));

    DECLARE @DateTo date =
        DATEADD(
            day,
            -(DATEDIFF(day, '19000103', @AvailableDate) % 7),
            @AvailableDate
        );

    DECLARE @DateFrom date;
    DECLARE @i int = 0;

    --------------------------------------------------------
    -- Промежуточные таблицы только для одной недели
    --------------------------------------------------------

    DECLARE @Stage TABLE
    (
        SegmentCode     tinyint,
        Bucket          int,
        CON_ID          bigint,
        CLI_ID          bigint,
        DtOpen          date,
        VolumeRub       decimal(38,2),
        WeightedRateSum decimal(38,10)
    );

    DECLARE @History TABLE
    (
        CLI_ID bigint NOT NULL,
        CON_ID bigint NOT NULL,
        DtOpen date NOT NULL,

        PRIMARY KEY (CLI_ID, CON_ID)
    );

    DECLARE @PriorClients TABLE
    (
        CLI_ID bigint PRIMARY KEY
    );

    DECLARE @SpecialOpen TABLE
    (
        CLI_ID     bigint NOT NULL,
        CON_ID     bigint NOT NULL,
        DtOpen     date NOT NULL,
        OpenAmount decimal(28,6),

        PRIMARY KEY (CLI_ID, DtOpen, CON_ID)
    );

    DECLARE @SpecialDays TABLE
    (
        CLI_ID          bigint NOT NULL,
        DtOpen          date NOT NULL,
        TotalOpenAmount decimal(38,6),

        PRIMARY KEY (CLI_ID, DtOpen)
    );

    DECLARE @NewMoney TABLE
    (
        CLI_ID bigint NOT NULL,
        CON_ID bigint NOT NULL,
        DtOpen date NOT NULL,

        PRIMARY KEY (CLI_ID, DtOpen, CON_ID)
    );

    --------------------------------------------------------
    -- БАТЧИ: одна итерация = одна неделя
    --------------------------------------------------------

    WHILE @i < @Weeks
    BEGIN

        SET @DateFrom = DATEADD(day, -6, @DateTo);

        DELETE FROM @Stage;
        DELETE FROM @History;
        DELETE FROM @PriorClients;
        DELETE FROM @SpecialOpen;
        DELETE FROM @SpecialDays;
        DELETE FROM @NewMoney;

        ----------------------------------------------------
        -- 1. Балансовый срез на последнюю дату недели
        ----------------------------------------------------

        ;WITH src AS
        (
            SELECT
                v.CON_ID,
                v.CLI_ID,
                v.OUT_RUB,
                v.PROD_NAME_res,
                v.termdays,
                v.rate_con,
                v.conv,
                CAST(v.dt_open AS date) AS DtOpen,

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
                    )
                    THEN 1 ELSE 0
                END AS is_mp,

                CASE
                    WHEN v.termdays BETWEEN 45 AND 70
                     AND ISNULL(v.conv, 'AT_THE_END')
                         = 'AT_THE_END'
                     AND ABS(v.rate_con - 0.1500) <= 0.0005
                    THEN 1 ELSE 0
                END AS special_2m,

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

            FROM ALM.ALM.vw_balance_rest_all v WITH (NOLOCK)

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
        ),

        ----------------------------------------------------
        -- 2. Классификация обычных вкладов
        -- Не обращаемся к надбавкам для МП, АП и промо 15%
        ----------------------------------------------------

        classified AS
        (
            SELECT
                s.CON_ID,
                s.CLI_ID,
                s.DtOpen,
                s.Bucket,
                s.OUT_RUB,
                s.rate_con,

                CASE
                    WHEN s.is_mp = 1 THEN 6

                    WHEN s.PROD_NAME_res = N'Активный процент'
                        THEN 5

                    WHEN s.special_2m = 1 THEN 0

                    WHEN f.retention = 1 THEN 2

                    WHEN f.is_pns_mpl = 1 THEN 7

                    WHEN f.is_nov = 1
                     AND d.dict_match = 1
                        THEN 4

                    WHEN f.is_ndp_ndm = 1
                     AND d.dict_match = 1
                        THEN 3

                    WHEN f.is_nov = 1
                      OR f.is_ndp_ndm = 1
                      OR d.dict_match = 1
                        THEN 2

                    ELSE 1
                END AS SegmentCode

            FROM src s

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
                  AND s.is_mp = 0
                  AND s.special_2m = 0
                  AND ISNULL(s.PROD_NAME_res, N'')
                      <> N'Активный процент'

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
                    END AS is_nov,

                    CASE
                        WHEN TRY_CAST(a.[НДП] AS int) = 1
                          OR TRY_CAST(a.[НДМ] AS int) = 1
                        THEN 1 ELSE 0
                    END AS is_ndp_ndm,

                    CASE
                        WHEN TRY_CAST(a.[Пнс] AS int) = 1
                          OR TRY_CAST(a.[Мпл] AS int) = 1
                        THEN 1 ELSE 0
                    END AS is_pns_mpl,

                    CASE
                        WHEN TRY_CAST(a.[От1] AS int) = 1
                          OR TRY_CAST(a.[Пк2] AS int) = 1
                        THEN 1 ELSE 0
                    END AS retention
            ) f

            OUTER APPLY
            (
                SELECT TOP (1)
                    1 AS dict_match

                FROM ALM_TEST.WORK.promo_new_money_rate_dict d
                    WITH (NOLOCK)

                WHERE s.is_mp = 0
                  AND s.special_2m = 0

                  AND ISNULL(s.PROD_NAME_res, N'')
                      <> N'Активный процент'

                  AND f.retention = 0
                  AND f.is_pns_mpl = 0

                  AND s.DtOpen
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

        prepared AS
        (
            SELECT
                SegmentCode,
                Bucket,

                CASE WHEN SegmentCode = 0
                    THEN CON_ID END AS CON_ID,

                CASE WHEN SegmentCode = 0
                    THEN CLI_ID END AS CLI_ID,

                CASE WHEN SegmentCode = 0
                    THEN DtOpen END AS DtOpen,

                CAST(OUT_RUB AS decimal(24,2)) AS VolumeRub,

                CAST(OUT_RUB AS decimal(24,2))
                    * CAST(rate_con AS decimal(12,8))
                    AS WeightedRate

            FROM classified
        )

        ----------------------------------------------------
        -- Обычные вклады агрегируем сразу.
        -- Детализацию оставляем только для промо 15%.
        ----------------------------------------------------

        INSERT INTO @Stage
        SELECT
            SegmentCode,
            Bucket,
            CON_ID,
            CLI_ID,
            DtOpen,
            SUM(VolumeRub),
            SUM(WeightedRate)

        FROM prepared

        GROUP BY
            SegmentCode,
            Bucket,
            CON_ID,
            CLI_ID,
            DtOpen

        OPTION (MAXDOP 1, RECOMPILE);

        ----------------------------------------------------
        -- 3. История только клиентов специального промо
        ----------------------------------------------------

        IF EXISTS
        (
            SELECT 1
            FROM @Stage
            WHERE SegmentCode = 0
              AND CON_ID IS NOT NULL
              AND CLI_ID IS NOT NULL
        )
        BEGIN

            INSERT INTO @History
            SELECT
                h.CLI_ID,
                h.CON_ID,
                MIN(CAST(h.DT_OPEN AS date))

            FROM LIQUIDITY.liq.DepositContract_all h
                WITH (NOLOCK)

            INNER JOIN
            (
                SELECT DISTINCT CLI_ID
                FROM @Stage
                WHERE SegmentCode = 0
                  AND CLI_ID IS NOT NULL
                  AND CON_ID IS NOT NULL
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

            GROUP BY
                h.CLI_ID,
                h.CON_ID

            OPTION (MAXDOP 1, RECOMPILE);

            ------------------------------------------------
            -- 4. Клиенты с историей до начала недели
            ------------------------------------------------

            INSERT INTO @PriorClients
            SELECT DISTINCT CLI_ID
            FROM @History
            WHERE DtOpen < @DateFrom;

            ------------------------------------------------
            -- 5. Остаток промо-вкладов на дату открытия
            -- Только для существующих клиентов
            ------------------------------------------------

            INSERT INTO @SpecialOpen
            SELECT
                s.CLI_ID,
                s.CON_ID,
                s.DtOpen,

                CAST(
                    ISNULL(b.OpenBalance, 0)
                    AS decimal(28,6)
                )

            FROM @Stage s

            INNER JOIN @PriorClients p
                ON p.CLI_ID = s.CLI_ID

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

            WHERE s.SegmentCode = 0
              AND s.CON_ID IS NOT NULL

            OPTION (MAXDOP 1, RECOMPILE);

            ------------------------------------------------
            -- 6. Объёмы открытий клиента по датам
            ------------------------------------------------

            INSERT INTO @SpecialDays
            SELECT
                CLI_ID,
                DtOpen,
                SUM(OpenAmount)

            FROM @SpecialOpen

            GROUP BY CLI_ID, DtOpen;

            ------------------------------------------------
            -- 7. Определение новых денег по истории остатков
            ------------------------------------------------

            ;WITH analysis_months AS
            (
                SELECT
                    CLI_ID,

                    DATEFROMPARTS(
                        YEAR(DtOpen),
                        MONTH(DtOpen),
                        1
                    ) AS month_start,

                    MAX(DtOpen) AS max_open

                FROM @SpecialDays

                GROUP BY
                    CLI_ID,
                    YEAR(DtOpen),
                    MONTH(DtOpen)
            ),

            analysis_windows AS
            (
                SELECT
                    CLI_ID,
                    month_start,
                    DATEADD(month, -1, month_start)
                        AS prev_start,
                    max_open

                FROM analysis_months
            ),

            saldo_clipped AS
            (
                SELECT
                    w.CLI_ID,
                    w.month_start,
                    w.max_open,

                    CAST(s.OUT_RUB AS decimal(28,6))
                        AS out_rub,

                    b.effective_from,
                    b.effective_to

                FROM analysis_windows w

                INNER JOIN @History h
                    ON h.CLI_ID = w.CLI_ID
                   AND h.DtOpen <= w.max_open

                INNER JOIN LIQUIDITY.liq.DepositContract_Saldo s
                    WITH (NOLOCK)

                    ON s.CON_ID = h.CON_ID

                   AND s.DT_FROM <
                       DATEADD(day, 1, w.max_open)

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
                        FROM
                        (
                            VALUES
                                (CAST(s.DT_FROM AS date)),
                                (h.DtOpen),
                                (w.prev_start)
                        ) v(dt)

                    ) AS effective_from,

                    (
                        SELECT MIN(v.dt)
                        FROM
                        (
                            VALUES
                                (
                                    ISNULL(
                                        CAST(s.DT_TO AS date),
                                        w.max_open
                                    )
                                ),
                                (w.max_open)
                        ) v(dt)

                    ) AS effective_to
                ) b

                WHERE s.OUT_RUB IS NOT NULL
                  AND b.effective_from <= b.effective_to
            ),

            saldo_events AS
            (
                SELECT
                    s.CLI_ID,
                    s.month_start,
                    e.event_date,
                    e.delta

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
                        END,

                        -s.out_rub
                    )
                ) e(event_date, delta)

                WHERE e.event_date IS NOT NULL

                UNION ALL

                SELECT
                    CLI_ID,
                    month_start,
                    prev_start,
                    CAST(0 AS decimal(28,6))

                FROM analysis_windows

                UNION ALL

                SELECT
                    CLI_ID,
                    month_start,
                    month_start,
                    CAST(0 AS decimal(28,6))

                FROM analysis_windows

                UNION ALL

                SELECT
                    CLI_ID,

                    DATEFROMPARTS(
                        YEAR(DtOpen),
                        MONTH(DtOpen),
                        1
                    ),

                    DtOpen,
                    CAST(0 AS decimal(28,6))

                FROM @SpecialDays
            ),

            event_agg AS
            (
                SELECT
                    CLI_ID,
                    month_start,
                    event_date,
                    SUM(delta) AS delta

                FROM saldo_events

                GROUP BY
                    CLI_ID,
                    month_start,
                    event_date
            ),

            timeline AS
            (
                SELECT
                    CLI_ID,
                    month_start,
                    event_date,

                    SUM(delta) OVER
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
                    CLI_ID,
                    month_start,
                    event_date,
                    balance,

                    MAX
                    (
                        CASE
                            WHEN event_date < month_start
                            THEN balance
                        END
                    ) OVER
                    (
                        PARTITION BY CLI_ID, month_start
                    ) AS prev_month_peak,

                    MAX
                    (
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
                    d.CLI_ID,
                    d.DtOpen,
                    d.TotalOpenAmount,

                    ISNULL(t.balance, 0) AS balance_today,

                    CASE
                        WHEN ISNULL(t.prev_month_peak, 0)
                          >= ISNULL(t.current_month_peak_before, 0)

                        THEN ISNULL(t.prev_month_peak, 0)

                        ELSE ISNULL(t.current_month_peak_before, 0)
                    END AS comparison_peak

                FROM @SpecialDays d

                LEFT JOIN timeline_stats t
                    ON t.CLI_ID = d.CLI_ID

                   AND t.month_start =
                       DATEFROMPARTS(
                           YEAR(d.DtOpen),
                           MONTH(d.DtOpen),
                           1
                       )

                   AND t.event_date = d.DtOpen
            ),

            day_budget AS
            (
                SELECT
                    CLI_ID,
                    DtOpen,
                    TotalOpenAmount,

                    CASE
                        WHEN TotalOpenAmount <= 0 THEN 0

                        WHEN balance_today <= comparison_peak
                            THEN 0

                        WHEN balance_today - comparison_peak
                             >= TotalOpenAmount
                            THEN TotalOpenAmount

                        ELSE balance_today - comparison_peak
                    END AS NewMoneyBudget

                FROM day_balances
            )

            ------------------------------------------------
            -- Частично новые и полностью новые деньги
            -- попадают в одну итоговую категорию.
            -- Сохраняем только подходящие договоры.
            ------------------------------------------------

            INSERT INTO @NewMoney
            SELECT
                s.CLI_ID,
                s.CON_ID,
                s.DtOpen

            FROM @SpecialOpen s

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

        END;

        ----------------------------------------------------
        -- 8. Финальная классификация и агрегация недели
        ----------------------------------------------------

        ;WITH final_classified AS
        (
            SELECT
                CASE
                    WHEN s.SegmentCode <> 0
                        THEN s.SegmentCode

                    WHEN s.CLI_ID IS NOT NULL
                     AND p.CLI_ID IS NULL
                        THEN 4

                    WHEN nm.CON_ID IS NOT NULL
                        THEN 3

                    ELSE 2
                END AS SegmentCode,

                s.Bucket,
                s.VolumeRub,
                s.WeightedRateSum

            FROM @Stage s

            LEFT JOIN @PriorClients p
                ON p.CLI_ID = s.CLI_ID

            LEFT JOIN @NewMoney nm
                ON nm.CON_ID = s.CON_ID
               AND nm.CLI_ID = s.CLI_ID
               AND nm.DtOpen = s.DtOpen
        ),

        agg AS
        (
            SELECT
                SegmentCode,

                GROUPING(SegmentCode) AS IsTotal,

                SUM(VolumeRub) AS VolumeRub,

                SUM(WeightedRateSum) AS WeightedRateSum,

                SUM(VolumeRub * Bucket) AS WeightedTermSum

            FROM final_classified

            GROUP BY GROUPING SETS
            (
                (SegmentCode),
                ()
            )

            HAVING COUNT_BIG(*) > 0
        )

        INSERT INTO @Result
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

            -- Объём в млрд рублей
            CAST(
                VolumeRub / 1000000000.0
                AS decimal(18,6)
            ),

            -- Доля от всего привлечения за неделю
            CAST(
                VolumeRub /
                NULLIF(
                    MAX(
                        CASE
                            WHEN IsTotal = 1 THEN VolumeRub
                        END
                    ) OVER (),
                    0
                )
                AS decimal(18,6)
            ),

            -- Средневзвешенная клиентская ставка
            CAST(
                WeightedRateSum / NULLIF(VolumeRub, 0)
                AS decimal(9,6)
            ),

            -- Средневзвешенный срок по бакетам
            CAST(
                WeightedTermSum / NULLIF(VolumeRub, 0)
                AS decimal(12,2)
            )

        FROM agg

        OPTION (MAXDOP 1, RECOMPILE);

        ----------------------------------------------------
        -- 9. Следующий недельный батч
        ----------------------------------------------------

        SET @DateTo = DATEADD(day, -7, @DateTo);
        SET @i = @i + 1;

    END;

    RETURN;
END;
GO


SELECT *
FROM ALM_TEST.reports.
    [fn_NewAttractionVolumes_forDimaMASS_Weekly_v2]
    ('20261125', 12)
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
