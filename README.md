WITH base AS (
    SELECT
        CASE
            WHEN INSTR(UPPER(region_name), 'МОСКВА') > 0
              OR INSTR(UPPER(region_name), 'МОСКОВСКАЯ ОБЛАСТЬ') > 0
                THEN 'Москва и МО'

            WHEN INSTR(UPPER(region_name), 'САНКТ-ПЕТЕРБУРГ') > 0
              OR INSTR(UPPER(region_name), 'ЛЕНИНГРАДСКАЯ ОБЛАСТЬ') > 0
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
        END AS pv_bucket_order,

        CASE
            WHEN vl_rub_fact <= 8000000
                THEN 'До 8 млн включительно'

            WHEN vl_rub_fact > 8000000
             AND vl_rub_fact <= 12000000
                THEN 'Свыше 8 до 12 млн включительно'

            WHEN vl_rub_fact > 12000000
             AND vl_rub_fact <= 18000000
                THEN 'Свыше 12 до 18 млн включительно'

            WHEN vl_rub_fact > 18000000
                THEN 'Свыше 18 млн'

            ELSE 'Пустой размер кредита'
        END AS loan_amount_bucket,

        CASE
            WHEN vl_rub_fact <= 8000000  THEN 1
            WHEN vl_rub_fact <= 12000000 THEN 2
            WHEN vl_rub_fact <= 18000000 THEN 3
            WHEN vl_rub_fact > 18000000  THEN 4
            ELSE 5
        END AS loan_amount_bucket_order
    FROM base
)
SELECT
    region_group,
    pv_bucket,
    loan_amount_bucket,
    SUM(vl_rub_fact) AS sum_vl_rub_fact
FROM bucketed
GROUP BY
    region_group,
    pv_bucket,
    pv_bucket_order,
    loan_amount_bucket,
    loan_amount_bucket_order
ORDER BY
    CASE region_group
        WHEN 'Москва и МО' THEN 1
        WHEN 'СПБ и ЛО'    THEN 2
        ELSE 3
    END,
    pv_bucket_order,
    loan_amount_bucket_order;
