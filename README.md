WITH target_clients AS (
    SELECT DISTINCT cli_id
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-07-30'
        AND section_name = 'Накопительный счёт'
        AND od_flag = 1
        AND MONTH(dt_open) IN (5, 6)
),
data_0208 AS (
    SELECT *
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-08-02'
        AND od_flag = 1
        AND section_name IN ('Накопительный счёт', 'Срочные')
        AND cli_id IN (SELECT cli_id FROM target_clients)
)
SELECT
    -- Объёмы
    SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) AS ns_out_rub_0208,
    SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) AS dep_out_rub_0208,

    -- Ставки
    CASE 
        WHEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) > 0
        THEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB * con_rate ELSE 0 END) /
             NULLIF(SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END), 0)
    END AS ns_avg_rate_0208,

    CASE 
        WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
        THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * con_rate ELSE 0 END) /
             NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
    END AS dep_avg_rate_0208,

    -- Срочность
    CASE 
        WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
        THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * termdays ELSE 0 END) /
             NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
    END AS dep_avg_termdays_0208
FROM data_0208
