USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @as_of    date = '2025-08-29';   -- дата среза
DECLARE @prod_id  int  = 654;            -- НС FIX

;WITH aug_clients AS (      -- клиенты, у кого есть НС, открытые в августе-2025
    SELECT DISTINCT t.cli_id
    FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
    WHERE t.dt_open >= '2025-08-01' AND t.dt_open < '2025-09-01'
      AND t.prod_id = @prod_id
      AND t.section_name = N'Накопительный счёт'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag = 1 AND t.cur = '810'
),
july_cons AS (              -- их «июльские» договоры (независимо от dt_rep)
    SELECT DISTINCT t.cli_id, t.con_id, t.TSEGMENTNAME
    FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
    JOIN aug_clients a ON a.cli_id = t.cli_id
    WHERE t.dt_open >= '2025-07-01' AND t.dt_open < '2025-08-01'
      AND t.prod_id = @prod_id
      AND t.section_name = N'Накопительный счёт'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag = 1 AND t.cur = '810'
)
-- ============== ТОТАЛ ==============
SELECT
    dt_rep              = @as_of,
    clients_cnt         = COUNT(DISTINCT j.cli_id),
    july_accounts_cnt   = COUNT(DISTINCT j.con_id),
    out_rub_total       = SUM(CAST(b.out_rub AS decimal(20,2)))
FROM july_cons j
JOIN ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
  ON b.con_id = j.con_id AND b.dt_rep = @as_of
WHERE b.out_rub IS NOT NULL;

-- ============== РАЗБИВКА ПО СЕГМЕНТАМ ==============
SELECT
    dt_rep            = @as_of,
    j.TSEGMENTNAME,
    clients_cnt       = COUNT(DISTINCT j.cli_id),
    july_accounts_cnt = COUNT(DISTINCT j.con_id),
    out_rub_total     = SUM(CAST(b.out_rub AS decimal(20,2)))
FROM july_cons j
JOIN ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
  ON b.con_id = j.con_id AND b.dt_rep = @as_of
WHERE b.out_rub IS NOT NULL
GROUP BY j.TSEGMENTNAME
ORDER BY j.TSEGMENTNAME;
