USE ALM_TEST;
GO

CREATE FUNCTION [alm_report].[fn_DepositFL_MonthlyGAP]
(
    @ReportDate date
)
RETURNS TABLE
AS
RETURN
(
    WITH months AS
    (
        SELECT
            n AS BucketNo,
            DATEADD(month, n,
                DATEFROMPARTS(
                    YEAR(@ReportDate),
                    MONTH(@ReportDate), 1
                )
            ) AS MonthStart
        FROM (VALUES
            (0),(1),(2),(3),(4),(5),
            (6),(7),(8),(9),(10),(11),
            (12),(13),(14),(15),(16),(17)
        ) t(n)
    ),

    agg AS
    (
        SELECT
            b.BucketNo,
            SUM(CAST(v.OUT_RUB AS decimal(24,2))) AS VolumeRub

        FROM ALM.ALM.vw_balance_rest_all v WITH (NOLOCK)

        CROSS APPLY
        (
            VALUES
            (
                CASE
                    WHEN DATEDIFF(month, @ReportDate, v.DT_CLOSE) >= 17
                        THEN 17
                    ELSE DATEDIFF(month, @ReportDate, v.DT_CLOSE)
                END
            )
        ) b(BucketNo)

        WHERE v.DT_REP = @ReportDate
          AND v.section_name = N'Срочные'
          AND v.block_name = N'Привлечение ФЛ'
          AND v.ap = N'Пассив'
          AND v.od_flag = 1
          AND v.cur = '810'
          AND v.OUT_RUB > 0
          AND v.DT_CLOSE >= @ReportDate

        GROUP BY b.BucketNo
    )

    SELECT
        m.MonthStart,

        CONCAT(
            CHOOSE(
                MONTH(m.MonthStart),
                N'Январь', N'Февраль', N'Март',
                N'Апрель', N'Май', N'Июнь',
                N'Июль', N'Август', N'Сентябрь',
                N'Октябрь', N'Ноябрь', N'Декабрь'
            ),
            N' ',
            YEAR(m.MonthStart),
            CASE WHEN m.BucketNo = 17 THEN N'+' ELSE N'' END
        ) AS MonthName,

        CAST(
            ISNULL(a.VolumeRub, 0) / 1000000000.0
            AS decimal(18,6)
        ) AS VolumeBln

    FROM months m
    LEFT JOIN agg a
        ON a.BucketNo = m.BucketNo
);
GO
