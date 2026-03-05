WITH
data AS (
    SELECT
        payment_period,
        GREATEST(con_rate, 0) AS con_rate_adj,   -- отрицательные → 0
        od_after_plan,
        od,
        premat_payment
    FROM cpr_report_new
    WHERE payment_period BETWEEN DATE '2024-01-01' AND DATE '2026-01-31'
      AND agg_prod_name IN ('Семейная ипотека', 'Льготная ипотека','ИТ ипотека','Дальневосточная ипотека')
),
bucketed AS (
    SELECT
        payment_period,
        FLOOR(con_rate_adj / 0.5) AS bucket_num,   -- номер интервала (0,1,2,...)
        od_after_plan,
        od,
        premat_payment
    FROM data
),
bucket_names AS (
    SELECT DISTINCT
        bucket_num,
        bucket_num * 0.5 AS low,
        (bucket_num + 1) * 0.5 AS high
    FROM bucketed
)
SELECT
    b.payment_period AS dt_rep,
    TO_CHAR(bn.low, 'FM9990.0') || '% – ' || TO_CHAR(bn.high, 'FM9990.0') || '%' AS bucket_name,
    SUM(b.od_after_plan) AS od_after_plan,
    SUM(b.od) AS od,
    SUM(b.premat_payment) AS premat,
    CASE
        WHEN SUM(b.od_after_plan) <= 0 THEN 0
        ELSE ROUND(100 * (1 - POWER(1 - SUM(b.premat_payment) / SUM(b.od_after_plan), 12)), 2)
    END AS CPR
FROM bucketed b
LEFT JOIN bucket_names bn ON b.bucket_num = bn.bucket_num
GROUP BY b.payment_period, b.bucket_num, bn.low, bn.high
ORDER BY b.payment_period, b.bucket_num;
