USE [ALM_TEST];
GO
SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

-- One-time installation. SQL Server resolves the real source types and collations.
-- No dynamic SQL is executed inside the resulting function.
-- Requires CREATE/ALTER FUNCTION and access to source metadata.
DECLARE @Types TABLE
(
    ColumnName sysname NOT NULL PRIMARY KEY,
    TypeSpec nvarchar(512) NULL
);

INSERT INTO @Types (ColumnName, TypeSpec)
SELECT
    name,
    system_type_name +
        CASE WHEN collation_name IS NULL THEN N''
             ELSE N' COLLATE ' + collation_name END
FROM sys.dm_exec_describe_first_result_set(
    N'SELECT CASE WHEN termdays BETWEEN 26 AND 44 THEN 31 ELSE termdays END AS SrcBucket, CLI_ID AS SrcCli, CON_ID AS SrcCon, conv AS SrcConv, PROD_NAME_res AS SrcProd, rate_con AS SrcRate, OUT_RUB AS SrcRub, termdays AS SrcTerm FROM ALM.ALM.vw_balance_rest_all;',
    NULL, 0
)
WHERE is_hidden = 0 AND error_number IS NULL;

INSERT INTO @Types (ColumnName, TypeSpec)
SELECT
    name,
    system_type_name +
        CASE WHEN collation_name IS NULL THEN N''
             ELSE N' COLLATE ' + collation_name END
FROM sys.dm_exec_describe_first_result_set(
    N'SELECT CLI_ID AS HistCli, CON_ID AS HistCon FROM LIQUIDITY.liq.DepositContract_all;',
    NULL, 0
)
WHERE is_hidden = 0 AND error_number IS NULL;

IF (SELECT COUNT(*) FROM @Types WHERE TypeSpec IS NOT NULL) <> 10
    THROW 50020, N'Не удалось получить типы полей источников. Функция не изменена.', 1;

-- Complete definition of the function:
DECLARE @Ddl nvarchar(max) = CAST(N'' AS nvarchar(max))
    + N'CREATE OR ALTER FUNCTION [reports].[fn_NewAttractionVolumes_forDimaMASS_Weekly_PQ]
(
    @AsOfDate date = NULL,
    @Weeks int = 9,
    @MergePnsMpl bit = 1,
    @Promo15From date = ''20261001'',
    @ClientBatchSize int = 3000,
    @DataLag int = 0,
    @VerifyFirstWeek bit = 0
)
RETURNS @W3_Result TABLE
(
    WeekStart date NULL,
    WeekEnd date NULL,
    Segment nvarchar(100) NULL,
    SegmentVolume decimal(18,6) NULL,
    SegmentSharePct decimal(18,4) NULL,
    ClientRate decimal(9,6) NULL,
    WeightedTerm decimal(12,2) NULL
)
AS
BEGIN
    -- A UDF cannot THROW or emit a diagnostic result set.
    -- Invalid parameters or a failed requested verification return an empty table.
    IF @DataLag IS NULL OR @DataLag < 0
       OR @ClientBatchSize IS NULL OR @ClientBatchSize <= 0
       OR @Promo15From IS NULL OR @VerifyFirstWeek IS NULL
       OR @MergePnsMpl IS NULL
        RETURN;

    SET @AsOfDate = ISNULL(@AsOfDate, CAST(GETDATE() AS date));
    DECLARE @AvailableDate date = DATEADD(day, -@DataLag, @AsOfDate);
    DECLARE @DateTo date =
        DATEADD(day, -((DATEDIFF(day, ''19000103'', @AvailableDate) % 7 + 7) % 7), @AvailableDate);
    DECLARE @DateFrom date;
    DECLARE @WeekNo int = 1;
    DECLARE @BatchNo int;
    DECLARE @BatchCount int;

    IF @Weeks IS NULL OR @Weeks NOT BETWEEN 1 AND 20
        RETURN;

    -- Staged tables; types of source columns are inherited at deployment.
    DECLARE @W3_Src TABLE
    (
        CON_ID {{SrcCon}} NULL,
        CLI_ID {{SrcCli}} NULL,
        OUT_RUB {{SrcRub}} NULL,
        PROD_NAME_res {{SrcProd}} NULL,
'
    + N'        termdays {{SrcTerm}} NULL,
        rate_con {{SrcRate}} NULL,
        conv {{SrcConv}} NULL,
        dt_open date NULL,
        is_mp int NULL,
        special_2m int NULL
    );

    DECLARE @W3_Special TABLE
    (
        CON_ID {{SrcCon}} NOT NULL,
        CLI_ID {{SrcCli}} NOT NULL,
        dt_open date NOT NULL,
        PRIMARY KEY CLUSTERED (CLI_ID, dt_open, CON_ID)
    );

    DECLARE @W3_History TABLE
    (
        CLI_ID {{HistCli}} NOT NULL,
        CON_ID {{HistCon}} NOT NULL,
        dt_open date NOT NULL,
        PRIMARY KEY CLUSTERED (CLI_ID, CON_ID)
    );

    DECLARE @W3_Prior TABLE
    (
        CLI_ID {{HistCli}} NOT NULL PRIMARY KEY CLUSTERED,
        BatchNo int NOT NULL,
        INDEX IX_PriorBatch NONCLUSTERED (BatchNo, CLI_ID)
    );

    DECLARE @W3_Open TABLE
    (
        CON_ID {{SrcCon}} NOT NULL,
        CLI_ID {{SrcCli}} NOT NULL,
        dt_open date NOT NULL,
        open_amount decimal(28,6) NULL,
        PRIMARY KEY CLUSTERED (CLI_ID, dt_open, CON_ID)
    );

    DECLARE @W3_Days TABLE
    (
        CLI_ID {{SrcCli}} NOT NULL,
        dt_open date NOT NULL,
        total_open_amount decimal(38,6) NULL,
        PRIMARY KEY CLUSTERED (CLI_ID, dt_open)
    );

    DECLARE @W3_Windows TABLE
    (
        CLI_ID {{SrcCli}} NOT NULL,
        month_start date NULL,
        prev_start date NULL,
        max_open date NULL
    );

    -- Not unique: retain the original join by CON_ID and all status rows.
    DECLARE @W3_Allocation TABLE
    (
        CON_ID {{SrcCon}} NOT NULL,
        new_money_status int NULL,
'
    + N'        INDEX IX_Allocation NONCLUSTERED (CON_ID, new_money_status)
    );

    DECLARE @W3_Agg TABLE
    (
        bucket_code {{SrcBucket}} NULL,
        segment_name nvarchar(100) COLLATE DATABASE_DEFAULT NULL,
        is_total tinyint NULL,
        segment_vol decimal(38,2) NULL,
        weighted_rate_sum decimal(38,10) NULL
    );

    DECLARE @W3_Raw TABLE
    (
        Bucket {{SrcBucket}} NULL,
        Segment nvarchar(100) COLLATE DATABASE_DEFAULT NULL,
        SegmentVolume decimal(18,6) NULL,
        SegmentSharePct decimal(18,4) NULL,
        ClientRate decimal(9,6) NULL
    );

    DECLARE @W3_Mapped TABLE
    (
        Bucket {{SrcBucket}} NULL,
        Segment nvarchar(100) COLLATE DATABASE_DEFAULT NULL,
        SegmentVolume decimal(18,6) NULL,
        SegmentSharePct decimal(18,4) NULL,
        ClientRate decimal(9,6) NULL
    );

    DECLARE @W3_Reference TABLE
    (
        Bucket {{SrcBucket}} NULL,
        Segment nvarchar(100) COLLATE DATABASE_DEFAULT NULL,
        SegmentVolume decimal(18,6) NULL,
        SegmentSharePct decimal(18,4) NULL,
        ClientRate decimal(9,6) NULL
    );

    WHILE @WeekNo <= @Weeks
    BEGIN
        SET @DateFrom = DATEADD(day, -6, @DateTo);

        -- 1. Snapshot, once per week

        ;WITH mp_list AS
        (
            SELECT DISTINCT prod_name_res
            FROM (VALUES
                  (N''ДОМа надёжно'')
                , (N''Всё в ДОМ'')
                , (N''Надёжный'')
                , (N''Надёжный VIP'')
                , (N''Надёжный премиум'')
                , (N''Надёжныйпромо'')
'
    + N'                , (N''Надёжный промо'')
                , (N''Надёжный старт'')
                , (N''Надёжный T2'')
                , (N''Надёжный Т2'')
                , (N''Надёжный Мегафон'')
                , (N''Надёжныйпрайм'')
                , (N''Надёжный прайм'')
                , (N''Надёжный процент'')
                , (N''Надёжныйпроцент'')
                , (N''Могучий'')
            ) m(prod_name_res)
        )

        INSERT INTO @W3_Src
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

            , CASE
                WHEN v.dt_open >= @Promo15From
                 AND v.termdays BETWEEN 45 AND 70
                 AND ISNULL(v.conv, ''AT_THE_END'') = ''AT_THE_END''
                 AND ABS(v.rate_con - 0.1500) <= 0.0005
                    THEN 1
                ELSE 0
              END AS special_2m

        FROM ALM.ALM.vw_balance_rest_all v WITH (NOLOCK)

        LEFT JOIN mp_list mp
            ON mp.prod_name_res = v.PROD_NAME_res

        WHERE v.DT_REP = @DateTo

          AND v.dt_open >= @DateFrom
          AND v.dt_open < DATEADD(day, 1, @DateTo)

          AND v.section_name = N''Срочные''
          AND v.block_name = N''Привлечение ФЛ''
          AND v.ap = N''Пассив''
          AND v.od_flag = 1
          AND v.cur = ''810''
          AND v.OUT_RUB > 0
'
    + N'          AND ISNULL(v.is_floatrate, 0) = 0
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
              OR v.TSegmentName <> N''ДЧБО''
          )

          AND
          (
              v.PROD_NAME_res IS NULL
              OR v.PROD_NAME_res NOT IN
              (
                  N''Классический'',
                  N''Привилегия'',
                  N''Достояние'',
                  N''Достояние без опций'',
                  N''Достояние с пополнением и снятием'',
                  N''Достояние (перем.ст.) без опций'',
                  N''Достояние (перем.ст.) с пополнением и снятием''
              )
          )

        OPTION (MAXDOP 1, RECOMPILE);

        -- 2. Promo contracts and client history

        INSERT INTO @W3_Special
        SELECT DISTINCT
              CON_ID
            , CLI_ID
            , dt_open

        FROM @W3_Src

        WHERE special_2m = 1
          AND is_mp = 0

          AND
          (
              PROD_NAME_res IS NULL
              OR PROD_NAME_res <> N''Активный процент''
          )

          AND CLI_ID IS NOT NULL
          AND CON_ID IS NOT NULL

        OPTION (MAXDOP 1, RECOMPILE);

        IF EXISTS (SELECT 1 FROM @W3_Special)
        BEGIN

            INSERT INTO @W3_History

            SELECT
                  h.CLI_ID
'
    + N'                , h.CON_ID
                , MIN(CAST(h.DT_OPEN AS date))

            FROM LIQUIDITY.liq.DepositContract_all h
                WITH (NOLOCK)

            INNER JOIN
            (
                SELECT DISTINCT CLI_ID
                FROM @W3_Special
            ) sc
                ON sc.CLI_ID = h.CLI_ID

            WHERE h.DT_OPEN IS NOT NULL
              AND h.DT_OPEN < DATEADD(day, 1, @DateTo)
              AND h.CON_ID IS NOT NULL
              AND h.cli_short_name = N''ФЛ''
              AND h.cur = ''RUR''

              AND
              (
                  h.PROD_NAME IS NULL
                  OR h.PROD_NAME NOT IN
                  (
                      N''Агентские эскроу ФЛ по ставке КС+спред'',
                      N''Спец. банк.счёт'',
                      N''Залоговый'',
                      N''Агентские эскроу ФЛ по ставке КС/2'',
                      N''Эскроу'',
                      N''Депозит ФЛ (суррогатный договор для счета Новой Афины)''
                  )
              )

            GROUP BY h.CLI_ID, h.CON_ID

            OPTION (MAXDOP 1, RECOMPILE);

        END;

        INSERT INTO @W3_Prior
        SELECT
              CLI_ID

            , CAST(
                (ROW_NUMBER() OVER (ORDER BY CLI_ID) - 1)
                / @ClientBatchSize + 1
                AS int
              ) AS BatchNo

        FROM @W3_History

        WHERE dt_open < @DateFrom

        GROUP BY CLI_ID

        OPTION (MAXDOP 1, RECOMPILE);

        SELECT @BatchCount = ISNULL(MAX(BatchNo), 0) FROM @W3_Prior;
        SET @BatchNo = 1;

'
    + N'        -- 3. Historical balances by batches

        WHILE @BatchNo <= @BatchCount
        BEGIN

            INSERT INTO @W3_Open
            SELECT
                  s.CON_ID
                , s.CLI_ID
                , s.dt_open

                , CAST(
                    ISNULL(b.open_balance, 0)
                    AS decimal(28,6)
                  ) AS open_amount

            FROM @W3_Special s

            INNER JOIN @W3_Prior p
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

            INSERT INTO @W3_Days
            SELECT
                  CLI_ID
                , dt_open
                , SUM(open_amount) AS total_open_amount

            FROM @W3_Open

            GROUP BY CLI_ID, dt_open

            OPTION (MAXDOP 1, RECOMPILE);

            INSERT INTO @W3_Windows
            SELECT
                  CLI_ID

                , DATEFROMPARTS(
                      YEAR(dt_open),
                      MONTH(dt_open),
'
    + N'                      1
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

            FROM @W3_Days

            GROUP BY
                  CLI_ID
                , YEAR(dt_open)
                , MONTH(dt_open)

            OPTION (MAXDOP 1, RECOMPILE);

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

                FROM @W3_Windows w

                INNER JOIN @W3_History h
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
'
    + N'                            SELECT MAX(v.dt)
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
'
    + N'                                THEN DATEADD(day, 1, s.effective_to)
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
                FROM @W3_Windows

                UNION ALL

                SELECT
                      CLI_ID
                    , month_start
                    , month_start
                    , CAST(0 AS decimal(28,6))
                FROM @W3_Windows

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
                FROM @W3_Days
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
'
    + N'                    , month_start
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
'
    + N'                      d.CLI_ID
                    , d.dt_open
                    , d.total_open_amount
                    , ISNULL(t.balance, 0) AS balance_today

                    , CASE
                        WHEN ISNULL(t.prev_month_peak, 0)
                           >= ISNULL(t.current_month_peak_before, 0)
                            THEN ISNULL(t.prev_month_peak, 0)

                        ELSE ISNULL(t.current_month_peak_before, 0)
                      END AS comparison_peak

                FROM @W3_Days d

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

            INSERT INTO @W3_Allocation

'
    + N'            SELECT
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

            FROM @W3_Open s

            INNER JOIN day_budget d
                ON d.CLI_ID = s.CLI_ID
               AND d.dt_open = s.dt_open

            OPTION (MAXDOP 1, RECOMPILE);

            DELETE FROM @W3_Open;
            DELETE FROM @W3_Days;
            DELETE FROM @W3_Windows;

            SET @BatchNo = @BatchNo + 1;
        END;

        DELETE FROM @W3_History;
        DELETE FROM @W3_Special;

        -- 4. Classify and reproduce original bucket-level rounding

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
'
    + N'                    AS special_new_money_status

            FROM @W3_Src s

            LEFT JOIN @W3_Prior p
                ON p.CLI_ID = s.CLI_ID

            LEFT JOIN @W3_Allocation sa
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
                          WHEN ISNULL(s.conv, ''AT_THE_END'')
                               = ''AT_THE_END''
                              THEN ''AT_THE_END''
                          ELSE ''NOT_AT_THE_END''
                      END

                  AND s.OUT_RUB
                      BETWEEN d.amount_from AND d.amount_to

'
    + N'                  AND ABS(s.rate_con - d.promo_rate) <= 0.0005

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
                        THEN N''Маркетплейсы''

                    WHEN m.PROD_NAME_res = N''Активный процент''
                        THEN N''Активный процент''

                    WHEN m.special_2m = 1
                     AND m.is_new_client = 1
                        THEN N''Новые клиенты''

'
    + N'                    WHEN m.special_2m = 1
                     AND m.special_new_money_status = 2
                        THEN N''Новые деньги''

                    WHEN m.special_2m = 1
                     AND m.special_new_money_status = 1
                        THEN N''Частично новые деньги''

                    WHEN m.special_2m = 1
                        THEN N''Другие вклады со ставкой новых денег''

                    WHEN m.retention = 1
                        THEN N''Автоматический механизм удержания''

                    WHEN m.is_nov = 1
                     AND m.dict_match = 1
                     AND m.is_pns_mpl = 0
                        THEN N''Новые клиенты''

                    WHEN m.is_ndp_ndm = 1
                     AND m.dict_match = 1
                    AND m.is_pns_mpl = 0
                        THEN N''Новые деньги''

                    WHEN (m.is_nov = 1 OR m.is_ndp_ndm = 1)
                     AND m.dict_match IS NULL
                     AND m.is_pns_mpl = 0
                        THEN N''Новые деньги без справочника''

                    WHEN m.dict_match = 1
                     AND m.is_pns_mpl = 0
                        THEN N''Другие вклады со ставкой новых денег''

                    WHEN m.is_pns_mpl = 1
                        THEN N''Вклады ПНС и МПЛ''

                    ELSE N''Остальные вклады''

                  END AS segment_name

                , CASE
                    WHEN m.termdays BETWEEN 26 AND 44 THEN 31
                    WHEN m.termdays BETWEEN 45 AND 70 THEN 61
'
    + N'                    WHEN m.termdays BETWEEN 85 AND 119 THEN 91
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

        INSERT INTO @W3_Agg
        SELECT
              bucket_code

            , CASE
                WHEN GROUPING(segment_name) = 1
                    THEN N''Итого''
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
'
    + N'            , (bucket_code)
        )

        OPTION (MAXDOP 1, RECOMPILE);

        DELETE FROM @W3_Src;
        DELETE FROM @W3_Prior;
        DELETE FROM @W3_Allocation;

        INSERT INTO @W3_Raw
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

        FROM @W3_Agg a

        CROSS JOIN
        (
            SELECT SUM(segment_vol) AS grand_total
            FROM @W3_Agg
            WHERE is_total = 0
        ) t;

        DELETE FROM @W3_Agg;

        -- 5. Optional comparison; off by default.

        IF @VerifyFirstWeek = 1 AND @WeekNo = 1 AND @DateFrom >= @Promo15From
        BEGIN

            INSERT INTO @W3_Reference
            SELECT
                  Bucket
                , Segment
                , SegmentVolume
                , SegmentSharePct
                , ClientRate

            FROM ALM_TEST.reports.
                [fn_NewAttractionVolumes_forDimaMASS_v1.6]
            (
                  @DateTo
                , @DateFrom
                , @DateTo
                , 0
            )

            OPTION (MAXDOP 1, RECOMPILE);

'
    + N'            IF EXISTS (SELECT * FROM @W3_Raw EXCEPT SELECT * FROM @W3_Reference)
            OR EXISTS (SELECT * FROM @W3_Reference EXCEPT SELECT * FROM @W3_Raw)
            BEGIN
                DELETE FROM @W3_Result;
                RETURN;
            END;
            DELETE FROM @W3_Reference;
        END;

        -- 6. Original report mapping, then weekly aggregation

        ;WITH mapped AS
        (
            SELECT
                Bucket,

                CASE
                    WHEN Segment = N''Остальные вклады''
                        THEN N''Базовая ставка''

                    WHEN Segment IN
                    (
                        N''Автоматический механизм удержания'',
                        N''Другие вклады со ставкой новых денег'',
                        N''Новые деньги без справочника''
                    )
                        THEN N''Удержание''

                    WHEN Segment IN
                    (
                        N''Частично новые деньги'',
                        N''Новые деньги''
                    )
                        THEN N''Новые деньги''

                    WHEN Segment = N''Вклады ПНС и МПЛ'' AND @MergePnsMpl = 1
                        THEN N''Базовая ставка''

                    WHEN Segment = N''Вклады ПНС и МПЛ''
                        THEN N''Пенсионеры и Маркетплейсы''

                    ELSE Segment

                END AS Segment,

                SegmentVolume,
                SegmentSharePct,
                ClientRate

            FROM @W3_Raw
        )

        INSERT INTO @W3_Mapped
        SELECT
'
    + N'              Bucket
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

        FROM mapped

        GROUP BY Bucket, Segment

        OPTION (MAXDOP 1, RECOMPILE);

        INSERT INTO @W3_Result
        SELECT
              @DateFrom
            , @DateTo
            , Segment

            , CAST(
                SUM(SegmentVolume)
                AS decimal(18,6)
              )

            , CAST(
                SUM(SegmentSharePct)
                AS decimal(18,4)
              )

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

            , CAST(
                CAST(
                    SUM(SegmentVolume * Bucket)
                    AS decimal(30,10)
                )
                / NULLIF(
'
    + N'                    CAST(SUM(SegmentVolume) AS decimal(28,6)),
                    0
                )
                AS decimal(12,2)
              )

        FROM @W3_Mapped

        GROUP BY Segment

        OPTION (MAXDOP 1, RECOMPILE);

        DELETE FROM @W3_Raw;
        DELETE FROM @W3_Mapped;

        SET @DateTo = DATEADD(day, -7, @DateTo);
        SET @WeekNo = @WeekNo + 1;
    END;
    RETURN;
END;
'
;

SELECT @Ddl = REPLACE(@Ddl, N'{{HistCli}}', TypeSpec) FROM @Types WHERE ColumnName = N'HistCli';
SELECT @Ddl = REPLACE(@Ddl, N'{{HistCon}}', TypeSpec) FROM @Types WHERE ColumnName = N'HistCon';
SELECT @Ddl = REPLACE(@Ddl, N'{{SrcBucket}}', TypeSpec) FROM @Types WHERE ColumnName = N'SrcBucket';
SELECT @Ddl = REPLACE(@Ddl, N'{{SrcCli}}', TypeSpec) FROM @Types WHERE ColumnName = N'SrcCli';
SELECT @Ddl = REPLACE(@Ddl, N'{{SrcCon}}', TypeSpec) FROM @Types WHERE ColumnName = N'SrcCon';
SELECT @Ddl = REPLACE(@Ddl, N'{{SrcConv}}', TypeSpec) FROM @Types WHERE ColumnName = N'SrcConv';
SELECT @Ddl = REPLACE(@Ddl, N'{{SrcProd}}', TypeSpec) FROM @Types WHERE ColumnName = N'SrcProd';
SELECT @Ddl = REPLACE(@Ddl, N'{{SrcRate}}', TypeSpec) FROM @Types WHERE ColumnName = N'SrcRate';
SELECT @Ddl = REPLACE(@Ddl, N'{{SrcRub}}', TypeSpec) FROM @Types WHERE ColumnName = N'SrcRub';
SELECT @Ddl = REPLACE(@Ddl, N'{{SrcTerm}}', TypeSpec) FROM @Types WHERE ColumnName = N'SrcTerm';

IF CHARINDEX(N'{{', @Ddl) > 0
    THROW 50021, N'Не все типы полей подставлены. Функция не изменена.', 1;

EXEC sys.sp_executesql @Ddl;
GO


SELECT *
FROM ALM_TEST.reports.fn_NewAttractionVolumes_forDimaMASS_Weekly_PQ
(DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT)
ORDER BY WeekEnd DESC, Segment;
