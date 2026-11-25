USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @ReportDate date = '20261118';

SELECT
    DATEFROMPARTS(
        YEAR(v.DT_CLOSE),
        MONTH(v.DT_CLOSE),
        1
    ) AS CloseMonth,

    CAST(
        SUM(CAST(v.OUT_RUB AS decimal(24,2)))
        / 1000000000.0
        AS decimal(18,6)
    ) AS VolumeBln

FROM ALM.ALM.vw_balance_rest_all v WITH (NOLOCK)

WHERE v.DT_REP = @ReportDate
  AND v.section_name = N'Срочные'
  AND v.block_name = N'Привлечение ФЛ'
  AND v.ap = N'Пассив'
  AND v.od_flag = 1
  AND v.cur = '810'
  AND v.OUT_RUB > 0

GROUP BY
    DATEFROMPARTS(
        YEAR(v.DT_CLOSE),
        MONTH(v.DT_CLOSE),
        1
    )

ORDER BY CloseMonth

OPTION (MAXDOP 1, RECOMPILE);
