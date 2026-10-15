DECLARE @dt_to date = DATEADD(day,-2,CAST(GETDATE() AS date));
DECLARE @dt_from date = DATEADD(day,-8,CAST(GETDATE() AS date));

SELECT *
FROM ALM_TEST.reports.fn_NewAttractionVolumes
(
    @dt_to,
    @dt_from,
    @dt_to,
    0
);
