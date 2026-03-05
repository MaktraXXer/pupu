WITH
base AS (
    SELECT
        payment_period,
        agg_prod_name,
        od_after_plan,
        od,
        premat_payment,
        GREATEST(con_rate, 0) AS rate_adj
    FROM cpr_report_new
    WHERE payment_period BETWEEN DATE '2024-01-01' AND DATE '2026-01-31'
      AND agg_prod_name IN ('Семейная ипотека', 'Льготная ипотека','ИТ ипотека','Дальневосточная ипотека')
),
-- общий od по каждому периоду
period_totals AS (
    SELECT
        payment_period,
        SUM(od) AS total_od
    FROM base
    GROUP BY payment_period
),
-- добавляем накопленную сумму od, упорядоченную по ставке
with_cum AS (
    SELECT
        b.*,
        pt.total_od,
        SUM(b.od) OVER (PARTITION BY b.payment_period ORDER BY b.rate_adj
                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cum_od
    FROM base b
    JOIN period_totals pt ON b.payment_period = pt.payment_period
),
-- назначаем номер бакета (0..9) по доле от общей od
bucketed AS (
    SELECT
        payment_period,
        rate_adj,
        od_after_plan,
        od,
        premat_payment,
        LEAST(FLOOR(cum_od / total_od * 10), 9) AS bucket_num   -- 10 групп
    FROM with_cum
),
-- вычисляем фактические границы ставок в каждом бакете для красивого названия
bucket_bounds AS (
    SELECT
        payment_period,
        bucket_num,
        MIN(rate_adj) AS min_rate,
        MAX(rate_adj) AS max_rate
    FROM bucketed
    GROUP BY payment_period, bucket_num
)
SELECT
    b.payment_period AS dt_rep,
    TO_CHAR(bb.min_rate, 'FM9990.0') || '% – ' || TO_CHAR(bb.max_rate, 'FM9990.0') || '%' AS bucket_name,
    SUM(b.od_after_plan) AS od_after_plan,
    SUM(b.od) AS od,
    SUM(b.premat_payment) AS premat,
    CASE
        WHEN SUM(b.od_after_plan) <= 0 THEN 0
        ELSE ROUND(100 * (1 - POWER(1 - SUM(b.premat_payment) / SUM(b.od_after_plan), 12)), 2)
    END AS CPR
FROM bucketed b
LEFT JOIN bucket_bounds bb ON b.payment_period = bb.payment_period AND b.bucket_num = bb.bucket_num
GROUP BY b.payment_period, b.bucket_num, bb.min_rate, bb.max_rate
ORDER BY b.payment_period, b.bucket_num;
