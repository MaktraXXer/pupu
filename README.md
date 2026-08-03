WITH base AS (
    SELECT
        region_name,
        vl_rub_fact,
        100 - ltv AS pv_pct
    FROM dm_common.cdm_sales_loan_mort
    WHERE dt_open      >= DATE '2026-01-01'
      AND dt_open_fact >= DATE '2026-01-01'
      AND lgot_program = 0
)
SELECT
    region_name,
    CASE
        WHEN pv_pct >= 0  AND pv_pct < 50 THEN '[0%-50%)'
        WHEN pv_pct >= 50 AND pv_pct < 70 THEN '[50%-70%)'
        WHEN pv_pct >= 70 AND pv_pct <= 80 THEN '[70%-80%]'
        WHEN pv_pct > 80                  THEN '>80%'
        ELSE 'Некорректный или пустой ПВ'
    END AS pv_bucket,
    SUM(vl_rub_fact) AS sum_vl_rub_fact
FROM base
GROUP BY
    region_name,
    CASE
        WHEN pv_pct >= 0  AND pv_pct < 50 THEN '[0%-50%)'
        WHEN pv_pct >= 50 AND pv_pct < 70 THEN '[50%-70%)'
        WHEN pv_pct >= 70 AND pv_pct <= 80 THEN '[70%-80%]'
        WHEN pv_pct > 80                  THEN '>80%'
        ELSE 'Некорректный или пустой ПВ'
    END
ORDER BY
    region_name,
    pv_bucket;
