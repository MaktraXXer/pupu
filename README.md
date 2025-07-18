WITH base AS (
    SELECT 
        c.cli_id,
        c.con_id,
        c.dt_open,
        s.out_rub,
        t.con_rate AS rate_balance
    FROM dds.contract c
    JOIN dds.con_rate t
      ON c.con_id = t.con_id 
     AND DATE '2025-07-16' BETWEEN t.dt_from AND t.dt_to
    JOIN dds.con_saldo s
      ON c.con_id = s.con_id 
     AND DATE '2025-07-16' BETWEEN s.dt_from AND s.dt_to
    WHERE c.prod_id = 654
),
flag_july_ns AS (
    SELECT DISTINCT c.cli_id,
           1 AS has_july_ns
    FROM dds.contract c
    WHERE c.prod_id = 654
      AND c.dt_open BETWEEN DATE '2025-07-01' AND DATE '2025-07-31'
)
SELECT 
    b.cli_id,
    b.con_id,
    b.dt_open,
    b.out_rub,
    b.rate_balance,
    NVL(f.has_july_ns, 0) AS has_july_ns
FROM base b
LEFT JOIN flag_july_ns f
       ON b.cli_id = f.cli_id
ORDER BY b.cli_id, b.dt_open;
