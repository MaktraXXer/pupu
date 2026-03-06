WITH
enriched AS (
    SELECT
        payment_period,
        agg_prod_name,
        od_after_plan,
        od,
        premat_payment,
        GREATEST(con_rate * 100, 0) AS pct,
        CASE
            WHEN GREATEST(con_rate * 100, 0) < 8.75 THEN
                FLOOR((GREATEST(con_rate * 100, 0) + 0.25) / 0.5) + 1
            WHEN GREATEST(con_rate * 100, 0) < 10 THEN 19
            WHEN GREATEST(con_rate * 100, 0) < 12 THEN 20
            WHEN GREATEST(con_rate * 100, 0) < 15 THEN 21
            WHEN GREATEST(con_rate * 100, 0) < 20 THEN 22
            ELSE 23
        END AS bucket_num
    FROM cpr_report_new
    WHERE payment_period BETWEEN DATE '2024-01-01' AND DATE '2026-01-31'
      AND agg_prod_name IN ('Семейная ипотека', 'Льготная ипотека','ИТ ипотека','Дальневосточная ипотека')
),
bucket_names AS (
    SELECT 1 AS bucket_num, 'A. 0% (0.0-0.25%)'   AS bucket_name FROM DUAL UNION ALL
    SELECT 2, 'B. 0.5% (0.25-0.75%)' FROM DUAL UNION ALL
    SELECT 3, 'C. 1.0% (0.75-1.25%)' FROM DUAL UNION ALL
    SELECT 4, 'D. 1.5% (1.25-1.75%)' FROM DUAL UNION ALL
    SELECT 5, 'E. 2.0% (1.75-2.25%)' FROM DUAL UNION ALL
    SELECT 6, 'F. 2.5% (2.25-2.75%)' FROM DUAL UNION ALL
    SELECT 7, 'G. 3.0% (2.75-3.25%)' FROM DUAL UNION ALL
    SELECT 8, 'H. 3.5% (3.25-3.75%)' FROM DUAL UNION ALL
    SELECT 9, 'I. 4.0% (3.75-4.25%)' FROM DUAL UNION ALL
    SELECT 10, 'J. 4.5% (4.25-4.75%)' FROM DUAL UNION ALL
    SELECT 11, 'K. 5.0% (4.75-5.25%)' FROM DUAL UNION ALL
    SELECT 12, 'L. 5.5% (5.25-5.75%)' FROM DUAL UNION ALL
    SELECT 13, 'M. 6.0% (5.75-6.25%)' FROM DUAL UNION ALL
    SELECT 14, 'N. 6.5% (6.25-6.75%)' FROM DUAL UNION ALL
    SELECT 15, 'O. 7.0% (6.75-7.25%)' FROM DUAL UNION ALL
    SELECT 16, 'P. 7.5% (7.25-7.75%)' FROM DUAL UNION ALL
    SELECT 17, 'Q. 8.0% (7.75-8.25%)' FROM DUAL UNION ALL
    SELECT 18, 'R. 8.5% (8.25-8.75%)' FROM DUAL UNION ALL
    SELECT 19, 'S. 8.75-10.0%' FROM DUAL UNION ALL
    SELECT 20, 'T. 10.0-12.0%' FROM DUAL UNION ALL
    SELECT 21, 'U. 12.0-15.0%' FROM DUAL UNION ALL
    SELECT 22, 'V. 15.0-20.0%' FROM DUAL UNION ALL
    SELECT 23, 'W. 20.0%+' FROM DUAL
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
