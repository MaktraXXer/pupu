WITH
enriched AS (
    SELECT
        payment_period,
        agg_prod_name,
        od_after_plan,
        od,
        premat_payment,
        con_rate * 100 AS pct,
        CASE
            WHEN con_rate * 100 < 0 THEN 0
            WHEN con_rate * 100 >= 15 THEN 23
            WHEN con_rate * 100 >= 12 THEN 22
            WHEN con_rate * 100 >= 11 THEN 21
            WHEN con_rate * 100 >= 10 THEN 20
            ELSE FLOOR(con_rate * 100 / 0.5)
        END AS bucket_num
    FROM cpr_report_new
    WHERE payment_period BETWEEN DATE '2024-01-01' AND DATE '2026-01-31'
      AND agg_prod_name IN ('Семейная ипотека', 'Льготная ипотека','ИТ ипотека','Дальневосточная ипотека')
),
bucket_names AS (
    SELECT
        bucket_num,
        CASE
            WHEN bucket_num <= 19 THEN 
                TO_CHAR(bucket_num + 1) || '. ' || 
                TO_CHAR(bucket_num * 0.5, 'FM990.0') || '% – ' || 
                TO_CHAR((bucket_num + 1) * 0.5, 'FM990.0') || '%'
            WHEN bucket_num = 20 THEN '21. 10-11%'
            WHEN bucket_num = 21 THEN '22. 11-12%'
            WHEN bucket_num = 22 THEN '23. 12-15%'
            WHEN bucket_num = 23 THEN '24. 15-20%'
        END AS bucket_name
    FROM (SELECT LEVEL - 1 AS bucket_num FROM dual CONNECT BY LEVEL <= 24)
)
SELECT
    e.payment_period AS dt_rep,
    bn.bucket_name,
    SUM(e.od_after_plan) AS od_after_plan,
    SUM(e.od) AS od,
    SUM(e.premat_payment) AS premat,
    CASE
        WHEN SUM(e.od_after_plan) <= 0 THEN 0
        ELSE ROUND(100 * (1 - POWER(1 - SUM(e.premat_payment) / SUM(e.od_after_plan), 12)), 2)
    END AS CPR
FROM enriched e
LEFT JOIN bucket_names bn ON e.bucket_num = bn.bucket_num
GROUP BY e.payment_period, bn.bucket_name, e.bucket_num
ORDER BY e.payment_period, e.bucket_num;
