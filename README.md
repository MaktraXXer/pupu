WITH base AS (
    SELECT
        CASE
            WHEN REGEXP_LIKE(region_name, 'Москва|Московская область', 'i')
                THEN 'Москва и МО'

            WHEN REGEXP_LIKE(
                     region_name,
                     'Санкт-Петербург|Ленинградская область',
                     'i'
                 )
                THEN 'СПБ и ЛО'

            ELSE 'Иные'
        END AS region_group,

        vl_rub_fact,
        100 - ltv AS pv_pct
    FROM dm_common.cdm_sales_loan_mort
    WHERE dt_open      >= DATE '2026-01-01'
      AND dt_open_fact >= DATE '2026-01-01'
      AND lgot_program = 0
),
bucketed AS (
    SELECT
        region_group,
        vl_rub_fact,
        CASE
            WHEN pv_pct >= 0  AND pv_pct < 20 THEN '[0%-20%)'
            WHEN pv_pct >= 20 AND pv_pct < 50 THEN '[20%-50%)'
            WHEN pv_pct >= 50 AND pv_pct < 70 THEN '[50%-70%)'
            WHEN pv_pct >= 70 AND pv_pct <= 80 THEN '[70%-80%]'
            WHEN pv_pct > 80                  THEN '>80%'
            ELSE 'Некорректный или пустой ПВ'
        END AS pv_bucket,

        CASE
            WHEN pv_pct >= 0  AND pv_pct < 20 THEN 1
            WHEN pv_pct >= 20 AND pv_pct < 50 THEN 2
            WHEN pv_pct >= 50 AND pv_pct < 70 THEN 3
            WHEN pv_pct >= 70 AND pv_pct <= 80 THEN 4
            WHEN pv_pct > 80                  THEN 5
            ELSE 6
        END AS pv_bucket_order
    FROM base
)
SELECT
    region_group,
    pv_bucket,
    SUM(vl_rub_fact) AS sum_vl_rub_fact
FROM bucketed
GROUP BY
    region_group,
    pv_bucket,
    pv_bucket_order
ORDER BY
    CASE region_group
        WHEN 'Москва и МО' THEN 1
        WHEN 'СПБ и ЛО'    THEN 2
        ELSE 3
    END,
    pv_bucket_order;
