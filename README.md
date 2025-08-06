/*  ► Параметры подставляет VBA ◄
    ► 124→122, 274→273, 550→548 — чтобы ровно 10 колонок ◄ */
DECLARE
    @ReportDate date = '{ReportDate}',
    @OpenFrom   date = '{OpenFrom}',
    @OpenTo     date = '{OpenTo}',
    @ExcludeMP  bit  = {ExcludeMP};

;WITH base AS (
    SELECT
        Bucket = CASE Bucket
                   WHEN 124 THEN 122
                   WHEN 274 THEN 273
                   WHEN 550 THEN 548
                   ELSE Bucket
                 END,
        Segment = CASE WHEN Segment = N'Итого'
                       THEN N'Общая структура' ELSE Segment END,
        Share   = CASE WHEN Segment = N'Общая структура'
                       THEN BucketSharePct ELSE SegmentSharePct END,
        Vol     = CASE WHEN Segment = N'Общая структура'
                       THEN BucketVolume   ELSE SegmentVolume   END
    FROM reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)
),
share_pivot AS (            -- проценты
    SELECT * FROM base
    PIVOT (SUM(Share) FOR Bucket IN
           ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
),
vol_pivot AS (              -- объёмы
    SELECT * FROM base
    PIVOT (SUM(Vol) FOR Bucket IN
           ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
)
SELECT 1 AS ord, Segment, * INTO #tmp FROM share_pivot
UNION ALL
SELECT 2, Segment + N' (объём)', * FROM vol_pivot;

SELECT Segment,
       [31],[61],[91],[122],[181],[273],[365],[548],[730],[1100]
FROM   #tmp
ORDER  BY ord, Segment;

DROP TABLE #tmp;
