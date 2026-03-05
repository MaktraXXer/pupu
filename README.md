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
            WHEN con_rate * 100 >= 20 THEN 29
            WHEN con_rate * 100 < 10 THEN FLOOR(con_rate * 100 / 0.5)
            ELSE 20 + FLOOR(con_rate * 100 - 10)
        END AS bucket_num
    FROM cpr_report_new
    WHERE payment_period BETWEEN DATE '2024-01-01' AND DATE '2026-01-31'
      AND agg_prod_name IN ('Семейная ипотека', 'Льготная ипотека','ИТ ипотека','Дальневосточная ипотека')
),
bucket_names AS (
    SELECT
        bucket_num,
        CASE
            WHEN bucket_num < 20 THEN TO_CHAR(bucket_num * 0.5, 'FM990.0') || '% – ' || TO_CHAR((bucket_num + 1) * 0.5, 'FM990.0') || '%'
            WHEN bucket_num = 29 THEN '19.0% – 20.0%'
            ELSE TO_CHAR(10 + (bucket_num - 20), 'FM990.0') || '% – ' || TO_CHAR(11 + (bucket_num - 20), 'FM990.0') || '%'
        END AS bucket_name
    FROM (SELECT LEVEL - 1 AS bucket_num FROM dual CONNECT BY LEVEL <= 30)
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
