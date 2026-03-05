WITH
data AS (
    SELECT
        payment_period,
        GREATEST(con_rate, 0) AS con_rate_adj,   -- отрицательные ставки → 0
        od_after_plan,
        od,
        premat_payment
    FROM cpr_report_new
    WHERE payment_period BETWEEN DATE '2024-01-01' AND DATE '2026-01-31'
      AND agg_prod_name IN ('Семейная ипотека', 'Льготная ипотека','ИТ ипотека','Дальневосточная ипотека')
),
global_max_rate AS (
    SELECT MAX(con_rate_adj) AS max_rate FROM data
),
-- генерация всех интервалов (номера бакетов и их границы) от 0 до максимальной ставки
intervals AS (
    SELECT 
        (LEVEL - 1) AS bucket_num,
        (LEVEL - 1) * 0.5 AS low,
        LEVEL * 0.5 AS high
    FROM dual
    CONNECT BY (LEVEL - 1) * 0.5 <= (SELECT max_rate FROM global_max_rate)
),
-- все уникальные периоды
periods AS (
    SELECT DISTINCT payment_period FROM data
),
-- декартово произведение периодов и интервалов
all_combinations AS (
    SELECT p.payment_period, i.bucket_num, i.low, i.high
    FROM periods p
    CROSS JOIN intervals i
),
-- агрегация данных по периодам и номерам бакетов
aggregated AS (
    SELECT
        payment_period,
        FLOOR(con_rate_adj / 0.5) AS bucket_num,
        SUM(od_after_plan) AS od_after_plan,
        SUM(od) AS od,
        SUM(premat_payment) AS premat
    FROM data
    GROUP BY payment_period, FLOOR(con_rate_adj / 0.5)
)
-- финальный результат со всеми комбинациями (пустые интервалы обнулены)
SELECT
    ac.payment_period AS dt_rep,
    TO_CHAR(ac.low, 'FM9990.0') || '% – ' || TO_CHAR(ac.high, 'FM9990.0') || '%' AS bucket_name,
    NVL(ag.od_after_plan, 0) AS od_after_plan,
    NVL(ag.od, 0) AS od,
    NVL(ag.premat, 0) AS premat,
    CASE
        WHEN NVL(ag.od_after_plan, 0) <= 0 THEN 0
        ELSE ROUND(100 * (1 - POWER(1 - NVL(ag.premat, 0) / NULLIF(ag.od_after_plan, 0), 12)), 2)
    END AS CPR
FROM all_combinations ac
LEFT JOIN aggregated ag ON ac.payment_period = ag.payment_period AND ac.bucket_num = ag.bucket_num
ORDER BY ac.payment_period, ac.bucket_num;
