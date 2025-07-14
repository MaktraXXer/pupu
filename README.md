/* --------------------------  настройки  -------------------------- */
DECLARE @cut_off_may date = '2025-05-31';   -- дата «конец мая 2025»

/* список продуктов фин-услуг (Main) */
DECLARE @MainProducts TABLE(prod_name_res nvarchar(100) PRIMARY KEY);
INSERT INTO @MainProducts VALUES
(N'Надёжный'),(N'Надёжный VIP'),(N'Надёжный премиум'),
(N'Надёжный промо'),(N'Надёжный старт'),
(N'Надёжный Т2'),(N'Надёжный Мегафон');

/* ---------- 1. клиенты, которые появлялись до 2025-01-01 ---------- */
WITH prev_cli AS (
    SELECT DISTINCT cli_id
    FROM   alm_test.dbo.fu_vintage_results_ext
    WHERE  dt_rep < '2025-01-01'
),

/* ---------- 2. «новички» 2025 года, пришедшие именно на ФУ ---------- */
new_fu_cli AS (
    SELECT DISTINCT f.cli_id
    FROM   alm_test.dbo.fu_vintage_results_ext f
    WHERE  f.dt_rep >= '2025-01-01'
      AND  f.prod_name_res IN (SELECT prod_name_res FROM @MainProducts)
      AND  NOT EXISTS (SELECT 1 FROM prev_cli p WHERE p.cli_id = f.cli_id)
),

/* ---------- 3. кто из них живой на 31-мая-2025 ---------- */
active_may AS (
    SELECT DISTINCT cli_id
    FROM   alm_test.dbo.fu_vintage_results_ext
    WHERE  dt_rep = @cut_off_may
      AND  cli_id IN (SELECT cli_id FROM new_fu_cli)
),

/* ---------- 4. классификация остатков на 31-мая-2025 ---------- */
classify AS (
    SELECT  a.cli_id,
            MAX(CASE WHEN f.prod_name_res IN (SELECT prod_name_res FROM @MainProducts)
                     THEN 1 END) AS has_fu,
            MAX(CASE WHEN f.prod_name_res NOT IN (SELECT prod_name_res FROM @MainProducts)
                     THEN 1 END) AS has_nonfu
    FROM   active_may         a
    JOIN   alm_test.dbo.fu_vintage_results_ext f
           ON f.cli_id = a.cli_id
          AND f.dt_rep = @cut_off_may
    GROUP BY a.cli_id
)

/* ---------- 5. итог ---------- */
SELECT
    (SELECT COUNT(*) FROM new_fu_cli)                               AS New_FU_clients_2025 ,
    (SELECT COUNT(*) FROM active_may)                               AS Active_as_of_2025_05_31 ,
    (SELECT COUNT(*) FROM classify WHERE has_fu     = 1 AND has_nonfu = 0) AS Only_FU_on_31_May ,
    (SELECT COUNT(*) FROM classify WHERE has_fu     = 0 AND has_nonfu = 1) AS Only_Bank_on_31_May ,
    (SELECT COUNT(*) FROM classify WHERE has_fu     = 1 AND has_nonfu = 1) AS Both_FU_and_Bank_on_31_May ;
