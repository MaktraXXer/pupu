
SELECT
    CASE
        WHEN GROUPING(promo_bucket) = 1 THEN N'ИТОГО 61 + 91 РК'
        ELSE CONCAT(promo_bucket, N' РК')
    END AS bucket,

    SUM(out_rub) AS volume,

    COUNT(DISTINCT con_id) AS contracts,

    COUNT(DISTINCT cli_id) AS clients,

    CAST(
        SUM(out_rub) * 1.0 /
        NULLIF(COUNT(DISTINCT cli_id), 0)
        AS decimal(20,2)
    ) AS avg_check

FROM #new
WHERE is_promo = 1
  AND promo_bucket IN (61, 91)

GROUP BY GROUPING SETS (
    (promo_bucket),
    ()
)

ORDER BY GROUPING(promo_bucket), promo_bucket;
