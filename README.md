USE ALM_TEST;
GO

CREATE OR ALTER FUNCTION
    [reports].[fn_NewAttractionVolumes_forDimaMASS_Weekly]
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

    -- Защита от слишком большого запроса
    IF @Weeks IS NULL OR @Weeks NOT BETWEEN 1 AND 12
        RETURN;

    DECLARE @AvailableDate date =
        DATEADD(day, -2, ISNULL(@AsOfDate, CAST(GETDATE() AS date)));

    -- Последняя доступная среда
    DECLARE @DateTo date =
        DATEADD(
            day,
            -(DATEDIFF(day, '19000103', @AvailableDate) % 7),
            @AvailableDate
        );

    DECLARE @DateFrom date;
    DECLARE @i int = 0;

    -- Один батч = одна неделя
    WHILE @i < @Weeks
    BEGIN

        SET @DateFrom = DATEADD(day, -6, @DateTo);

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
                        THEN N'Механизм удержания'

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
                ClientRate

            FROM ALM_TEST.reports.
                [fn_NewAttractionVolumes_forDimaMASS_v1.6]
            (
                @DateTo,
                @DateFrom,
                @DateTo,
                0
            )

            WHERE Segment <> N'Итого'
        ),

        agg AS
        (
            SELECT
                CASE
                    WHEN GROUPING(Segment) = 1
                        THEN N'Итого'
                    ELSE Segment
                END AS Segment,

                GROUPING(Segment) AS IsTotal,

                SUM(SegmentVolume) AS Volume,

                SUM(SegmentVolume * ClientRate)
                    / NULLIF(SUM(SegmentVolume), 0)
                    AS ClientRate,

                SUM(SegmentVolume * Bucket)
                    / NULLIF(SUM(SegmentVolume), 0)
                    AS WeightedTerm

            FROM mapped

            GROUP BY GROUPING SETS
            (
                (Segment),
                ()
            )

            HAVING COUNT_BIG(*) > 0
        )

        INSERT INTO @Result
        (
            WeekStart,
            WeekEnd,
            Segment,
            Volume,
            SharePct,
            ClientRate,
            WeightedTerm
        )
        SELECT
            @DateFrom,
            @DateTo,
            Segment,
            Volume,

            Volume / NULLIF(
                MAX(
                    CASE
                        WHEN IsTotal = 1 THEN Volume
                    END
                ) OVER (),
                0
            ) AS SharePct,

            ClientRate,
            WeightedTerm

        FROM agg

        OPTION (MAXDOP 1, RECOMPILE);

        -- Переход к следующему батчу
        SET @DateTo = DATEADD(day, -7, @DateTo);
        SET @i = @i + 1;

    END;

    RETURN;
END;
GO





SELECT *
FROM ALM_TEST.reports.
    [fn_NewAttractionVolumes_forDimaMASS_Weekly]('20261125', 12)
ORDER BY WeekEnd DESC, Segment;
