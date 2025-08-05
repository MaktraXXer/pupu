SELECT
    br.dt_rep,
    
    -- общий объём НС (все строки)
    SUM(br.OUT_RUB) AS ns_total_out_rub,

    -- объём только по строкам с ненулевой ставкой
    SUM(CASE WHEN br.con_rate IS NOT NULL THEN br.OUT_RUB ELSE 0 END) AS ns_out_rub_with_rate,

    -- средневзвешенная ставка по строкам с ненулевой ставкой
    CASE 
        WHEN SUM(CASE WHEN br.con_rate IS NOT NULL THEN br.OUT_RUB ELSE 0 END) > 0
        THEN SUM(CASE WHEN br.con_rate IS NOT NULL THEN br.OUT_RUB * br.con_rate ELSE 0 END)
             / SUM(CASE WHEN br.con_rate IS NOT NULL THEN br.OUT_RUB ELSE 0 END)
        ELSE NULL
    END AS ns_weighted_avg_rate

FROM ALM.vw_balance_rest_all br WITH (NOLOCK)
WHERE 
    br.section_name = 'Накопительный счёт'
    AND br.od_flag = 1
    AND br.dt_rep BETWEEN '2025-07-01' AND '2025-07-30'
    AND br.cli_id IN (
        SELECT DISTINCT cli_id
        FROM ALM.vw_balance_rest_all WITH (NOLOCK)
        WHERE 
            dt_rep = '2025-07-30'
            AND section_name = 'Накопительный счёт'
            AND od_flag = 1
            AND MONTH(dt_open) IN (5, 6)
    )
GROUP BY br.dt_rep
ORDER BY br.dt_rep
