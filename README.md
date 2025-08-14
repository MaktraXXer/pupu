-- … параметры и #fu остаются без изменений

/* 1) #seed31 */
WITH rows31 AS (
  SELECT
    t.cli_id, t.con_id,
    is_fu    = CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END,
    dt_close = CAST(t.dt_close AS date),
    vol31    = SUM(CASE WHEN t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
  FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
  LEFT JOIN #fu f ON f.prod_name_res = t.PROD_NAME_res
  WHERE t.dt_rep = @JulEOM
    AND t.block_name = N'Привлечение ФЛ' AND t.od_flag=1 AND t.cur='810'
    AND t.out_rub IS NOT NULL
    AND LTRIM(RTRIM(t.section_name)) IN (N'Срочные')
    AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  GROUP BY t.cli_id, t.con_id, CASE WHEN f.prod_name_res IS NULL THEN 0 ELSE 1 END, t.dt_close
),
-- … per_client / nonfu_close_aug без изменений, только там, где фильтруешь section_name:
-- AND LTRIM(RTRIM(t.section_name)) IN (N'Срочные')

/* 2) НС 31.07 и 12.08 */
SELECT
  t.cli_id,
  vol_ns31 = SUM(CASE WHEN t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #ns31
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id=t.cli_id
WHERE t.dt_rep = @JulEOM
  AND t.block_name = N'Привлечение ФЛ' AND t.od_flag=1 AND t.cur='810'
  AND t.out_rub IS NOT NULL
  AND LTRIM(RTRIM(t.section_name)) IN (N'Накопительный счёт')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.cli_id;

-- на 12.08 по договорам НС
SELECT
  t.con_id,
  t.cli_id,
  dt_open = CAST(t.dt_open AS date),
  vol_ns12 = SUM(CASE WHEN t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #ns12_con
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id=t.cli_id
WHERE t.dt_rep = @ChkDate
  AND t.block_name = N'Привлечение ФЛ' AND t.od_flag=1 AND t.cur='810'
  AND t.out_rub IS NOT NULL
  AND LTRIM(RTRIM(t.section_name)) IN (N'Накопительный счёт')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.con_id, t.cli_id, t.dt_open;

-- #ns12, #ns_open, #ns31_con, #ns_refill — как у тебя, без изменений.

/* 3) ОТКРЫТИЯ вкладов 01–12.08 */
SELECT
  t.cli_id,
  opened_fu   = MAX(CASE WHEN f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END),
  opened_bank = MAX(CASE WHEN f.prod_name_res IS NULL     THEN 1 ELSE 0 END),
  vol_fu      = SUM(CASE WHEN f.prod_name_res IS NOT NULL AND t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END),
  vol_bank    = SUM(CASE WHEN f.prod_name_res IS NULL     AND t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END)
INTO #open_aug
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id = t.cli_id
LEFT JOIN #fu  f ON f.prod_name_res = t.PROD_NAME_res
WHERE t.dt_rep = @ChkDate
  AND t.dt_open BETWEEN @CloseFrom AND @ChkDate
  AND t.block_name = N'Привлечение ФЛ' AND t.od_flag=1 AND t.cur='810'
  AND t.out_rub IS NOT NULL
  AND LTRIM(RTRIM(t.section_name)) IN (N'Срочные')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.cli_id;

-- принты — как у тебя; оставляю без изменения формул.

/* 6) мини-сверка, что разбивка клиентов корректна */
;WITH K AS (
  SELECT seed = (SELECT COUNT(*) FROM #seed31),
         bank_only = (SELECT COUNT(*) FROM #open_aug WHERE opened_bank=1 AND opened_fu=0),
         fu_only   = (SELECT COUNT(*) FROM #open_aug WHERE opened_fu=1 AND opened_bank=0),
         both      = (SELECT COUNT(*) FROM #open_aug WHERE opened_fu=1 AND opened_bank=1)
)
SELECT
  seed,
  bank_only, fu_only, both,
  no_opens = seed - (bank_only + fu_only + both)
FROM K;
