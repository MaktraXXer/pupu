SELECT DISTINCT
    cli_id,
    con_id,
    CASE 
        WHEN MONTH(dt_open) = 7 THEN 1 ELSE 0 
    END AS has_ns_july,
    CASE 
        WHEN MONTH(dt_open) = 8 THEN 1 ELSE 0 
    END AS has_ns_aug
FROM ALM.vw_balance_rest_all WITH (NOLOCK)
WHERE 
    od_flag = 1
    AND ap = 'Пассив'
    AND block_name = 'Привлечение ФЛ'
    AND section_name = 'Накопительный счёт'
    AND TSegmentName IN ('ДЧБО', 'Розничный бизнес')
    AND prod_id = '654'
    AND dt_rep BETWEEN '2025-07-27' AND '2025-08-02'
    AND dt_open >= '2025-07-01'
    AND MONTH(dt_open) IN (7, 8)
