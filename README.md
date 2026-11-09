SELECT *
FROM ALM_TEST.reports.fn_NewAttractionVolumes_forDimaPRIVATE_v1(
    '2026-11-04', '2026-10-20', '2026-11-04', 1
)
ORDER BY Bucket, Segment;
