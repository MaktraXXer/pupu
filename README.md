WITH active_ns AS (
    SELECT DISTINCT con_id
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        od_flag = 1
        AND ap = 'Пассив'
        AND block_name = 'Привлечение ФЛ'
        AND section_name = 'Накопительный счёт'
        AND TSegmentName IN ('ДЧБО', 'Розничный бизнес')
        AND prod_id = '654'
        AND dt_rep BETWEEN '2025-07-27' AND '2025-08-02'
),
ns_open_dates AS (
    SELECT 
        con_id,
        cli_id,
        MIN(dt_open) AS dt_open -- одна строка на клиента
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        od_flag = 1
        AND ap = 'Пассив'
        AND block_name = 'Привлечение ФЛ'
        AND section_name = 'Накопительный счёт'
        AND TSegmentName IN ('ДЧБО', 'Розничный бизнес')
        AND prod_id = '654'
        AND dt_open >= '2025-07-01' -- отсекаем всё старше июля
    GROUP BY con_id, cli_id
),
filtered_clients AS (
    SELECT 
        d.con_id,
        n.cli_id,
        CASE 
            WHEN MONTH(n.dt_open) = 7 THEN 1 
            ELSE 0 
        END AS has_ns_july,
        CASE 
            WHEN MONTH(n.dt_open) = 8 THEN 1 
            ELSE 0 
        END AS has_ns_aug
    FROM active_ns d
    JOIN ns_open_dates n ON d.con_id = n.con_id
)
SELECT *
FROM filtered_clients;
