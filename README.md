WITH target_con_ids AS (
    SELECT DISTINCT con_id
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-07-30'
        AND section_name = 'Накопительный счёт'
        AND od_flag = 1
        AND MONTH(dt_open) IN (5, 6)
),

daily_ns_data AS (
    SELECT
        dt_rep,
        OUT_RUB,
        con_rate
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE
        dt_rep BETWEEN '2025-07-01' AND '2025-07-30'
        AND section_name = 'Накопительный счёт'
        AND od_flag = 1
        AND con_id IN (SELECT con_id FROM target_con_ids)
)

SELECT
    dt_rep,

    -- Общий объём по НС (в том числе с null ставками)
    SUM(OUT_RUB) AS ns_total_out_rub,

    -- Объём только по строкам, где ставка есть
    SUM(CASE WHEN con_rate IS NOT NULL THEN OUT_RUB ELSE 0 END) AS ns_out_rub_with_rate,

    -- Средневзвешенная ставка ТОЛЬКО по строкам с ненулевыми ставками
    CASE 
        WHEN SUM(CASE WHEN con_rate IS NOT NULL THEN OUT_RUB ELSE 0 END) > 0
        THEN SUM(CASE WHEN con_rate IS NOT NULL THEN OUT_RUB * con_rate ELSE 0 END) 
             / SUM(CASE WHEN con_rate IS NOT NULL THEN OUT_RUB ELSE 0 END)
        ELSE NULL
    END AS ns_weighted_avg_rate

FROM daily_ns_data
GROUP BY dt_rep
ORDER BY dt_rep
