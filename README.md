SELECT *
FROM ALM_TEST.reports.fn_NewAttractionVolumes_forDimaPRIVATE_v1(
    '2026-11-04', '2026-10-20', '2026-11-04', 1
)
ORDER BY Bucket, Segment;


SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT *
FROM ALM_TEST.reports.fn_NewAttractionVolumes_forDimaMASS_v1.6(
    '2026-11-04',
    '2026-11-03',
    '2026-11-04',
    0
)
ORDER BY Bucket, Segment;

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
