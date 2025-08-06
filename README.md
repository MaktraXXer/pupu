WITH excluded_clients AS (
    SELECT DISTINCT cli_id
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-07-30'
        AND section_name = 'Накопительный счёт'
        AND od_flag = 1
        AND MONTH(dt_open) IN (5, 6)
        AND prod_id = '654'
),

clients_0730 AS (
    SELECT DISTINCT cli_id
    FROM ALM.vw_balance_rest_all a WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-07-30'
        AND od_flag = 1
        AND block_name = N'Привлечение ФЛ'
),

clients_0208 AS (
    SELECT DISTINCT cli_id
    FROM ALM.vw_balance_rest_all a WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-08-02'
        AND od_flag = 1
        AND block_name = N'Привлечение ФЛ'
),

new_clients_0208 AS (
    -- те, кто есть на 02.08, но НЕ было на 30.07
    SELECT a.cli_id
    FROM clients_0208 a
    LEFT JOIN clients_0730 b ON a.cli_id = b.cli_id
    WHERE b.cli_id IS NULL
),

target_clients AS (
    SELECT DISTINCT cli_id
    FROM (
        -- обычные (были 30.07 и не входят в excluded)
        SELECT a.cli_id
        FROM clients_0730 a
        LEFT JOIN excluded_clients b ON a.cli_id = b.cli_id
        WHERE b.cli_id IS NULL

        UNION

        -- новые клиенты
        SELECT cli_id FROM new_clients_0208
    ) all_clients
)
