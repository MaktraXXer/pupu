
USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @DateFrom date = CAST(DATEADD(day, -8, GETDATE()) AS date);
DECLARE @DateTo   date = CAST(DATEADD(day, -2, GETDATE()) AS date);

;WITH base AS (
    SELECT
        Bucket,
        Segment,
        SegmentVolume,
        SegmentSharePct,
        ClientRate,
        PartialNewMoneyVolume
    FROM ALM_TEST.reports.fn_NewAttractionVolumes_forDimaPRIVATE_v1(
        @DateTo,
        @DateFrom,
        @DateTo,
        1
    )
),

mapped AS (
    SELECT
        Bucket,

        CASE
            WHEN Segment = N'Базовая ставка или ниже'
                THEN N'Базовая ставка'

            WHEN Segment = N'Выше базовой ставки'
                THEN N'Индивидуальные согласования'

            WHEN Segment = N'Промоставка на действующие деньги'
                THEN N'Удержание'

            WHEN Segment IN (
                N'Новые деньги',
                N'Частично новые деньги'
            )
                THEN N'Новые деньги'

            ELSE Segment
        END AS Segment,

        SegmentVolume,
        SegmentSharePct,
        ClientRate,
        PartialNewMoneyVolume

    FROM base
)

SELECT
    Bucket,
    Segment,

    CAST(
        SUM(SegmentVolume)
        AS decimal(18,6)
    ) AS SegmentVolume,

    CAST(
        SUM(SegmentSharePct)
        AS decimal(18,4)
    ) AS SegmentSharePct,

    CAST(
        CASE
            WHEN COUNT(*) = 1
                THEN MAX(ClientRate)
            ELSE
                SUM(SegmentVolume * ClientRate)
                / NULLIF(SUM(SegmentVolume), 0)
        END
        AS decimal(9,6)
    ) AS ClientRate,

    CAST(
        SUM(PartialNewMoneyVolume)
        AS decimal(18,6)
    ) AS PartialNewMoneyVolume

FROM mapped
GROUP BY Bucket, Segment
ORDER BY Bucket, Segment;
