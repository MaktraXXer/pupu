SET NOCOUNT ON;

DECLARE @JulEOM    date = '2025-07-31';
DECLARE @ChkDate   date = '2025-08-12';
DECLARE @CloseFrom date = '2025-08-01';
DECLARE @CloseTo   date = '2025-08-11';  -- плановые закрытия (включительно)

-- ФУ-продукты (инлайн-список: быстрее, чем лишний JOIN)
-- N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
-- N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2',
-- N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный'

/* ===== 1) 31.07: агрегат по клиенту (вклады «Срочные» + НС) ===== */
IF OBJECT_ID('tempdb..#j31_dep') IS NOT NULL DROP TABLE #j31_dep;
SELECT
    t.cli_id,
    t.con_id,
    is_fu = CASE WHEN t.PROD_NAME_res IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                          N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                          N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                 THEN 1 ELSE 0 END,
    dt_close = t.dt_close,  -- datetime оставляем как есть (SARGable сравнение ниже)
    vol31 = SUM(CASE WHEN t.out_rub>0 THEN CONVERT(decimal(19,2),t.out_rub) ELSE 0 END)
INTO #j31_dep
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep = @JulEOM
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub IS NOT NULL
  AND t.section_name IN (N'Срочные',N'Срочные ')      -- без LTRIM/RTRIM для индексов
  AND (t.TSEGMENTNAME IS NULL OR t.TSEGMENTNAME IN (N'Розничный бизнес'))
GROUP BY t.cli_id, t.con_id, CASE WHEN t.PROD_NAME_res IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                                           N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                                           N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                  THEN 1 ELSE 0 END,
         t.dt_close;
CREATE CLUSTERED INDEX IX_j31_dep_cli ON #j31_dep(cli_id);

IF OBJECT_ID('tempdb..#ns31') IS NOT NULL DROP TABLE #ns31;
SELECT t.cli_id,
       vol_ns31 = SUM(CASE WHEN t.out_rub>0 THEN CONVERT(decimal(19,2),t.out_rub) ELSE 0 END)
INTO #ns31
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep = @JulEOM
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub IS NOT NULL
  AND t.section_name IN (N'Накопительный счёт',N'Накопительный счёт ')
  AND (t.TSEGMENTNAME IS NULL OR t.TSEGMENTNAME IN (N'Розничный бизнес'))
GROUP BY t.cli_id;
CREATE UNIQUE CLUSTERED INDEX IX_ns31_cli ON #ns31(cli_id);

-- клиенты с закрытиями НЕ-ФУ в августе (исключим)
IF OBJECT_ID('tempdb..#bad_nonfu_aug') IS NOT NULL DROP TABLE #bad_nonfu_aug;
SELECT DISTINCT t.cli_id
INTO #bad_nonfu_aug
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
WHERE t.section_name IN (N'Срочные',N'Срочные ')
  AND (t.TSEGMENTNAME IS NULL OR t.TSEGMENTNAME IN (N'Розничный бизнес'))
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.dt_close >= @CloseFrom AND t.dt_close < DATEADD(DAY,1,@CloseTo)
  AND t.PROD_NAME_res NOT IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                              N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                              N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный');
CREATE UNIQUE CLUSTERED INDEX IX_bad_cli ON #bad_nonfu_aug(cli_id);

/* SEED: только ФУ на 31.07, и все они закрываются 1–11.08, без НЕ-ФУ закрытий */
IF OBJECT_ID('tempdb..#seed31') IS NOT NULL DROP TABLE #seed31;
SELECT
    d.cli_id,
    vol_deposits_close31 = SUM(CASE WHEN d.is_fu=1 THEN d.vol31 ELSE 0 END),
    vol_ns31 = ISNULL(n.vol_ns31,0)
INTO #seed31
FROM #j31_dep d
LEFT JOIN #ns31 n ON n.cli_id = d.cli_id
LEFT JOIN #bad_nonfu_aug b ON b.cli_id = d.cli_id
GROUP BY d.cli_id, n.vol_ns31, b.cli_id
HAVING
    SUM(CASE WHEN d.is_fu=0 THEN 1 ELSE 0 END) = 0         -- на 31.07 нет НЕ-ФУ вкладов
AND SUM(CASE WHEN d.is_fu=1 THEN 1 ELSE 0 END) > 0         -- есть ФУ-вклады
AND SUM(CASE WHEN d.is_fu=1 AND d.dt_close >= @CloseFrom AND d.dt_close < DATEADD(DAY,1,@CloseTo) THEN 1 ELSE 0 END)
    = SUM(CASE WHEN d.is_fu=1 THEN 1 ELSE 0 END)           -- все ФУ закрываются в окне
AND b.cli_id IS NULL;                                      -- нет НЕ-ФУ закрытий в августе
CREATE UNIQUE CLUSTERED INDEX IX_seed_cli ON #seed31(cli_id);

/* ===== 2) 12.08: открытия вкладов (01–12.08) + НС объём ===== */
IF OBJECT_ID('tempdb..#dep1208') IS NOT NULL DROP TABLE #dep1208;
SELECT
    t.cli_id,
    dep_open_fu_flag   = MAX(CASE WHEN t.section_name IN (N'Срочные',N'Срочные ')
                                   AND t.PROD_NAME_res IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                                           N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                                           N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                   AND t.dt_open >= @CloseFrom AND t.dt_open < DATEADD(DAY,1,@ChkDate)
                                  THEN 1 ELSE 0 END),
    dep_open_bank_flag = MAX(CASE WHEN t.section_name IN (N'Срочные',N'Срочные ')
                                   AND t.PROD_NAME_res NOT IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                                               N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                                               N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                   AND t.dt_open >= @CloseFrom AND t.dt_open < DATEADD(DAY,1,@ChkDate)
                                  THEN 1 ELSE 0 END),
    vol_dep_fu_1208    = SUM(CASE WHEN t.section_name IN (N'Срочные',N'Срочные ')
                                   AND t.PROD_NAME_res IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                                           N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                                           N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                   AND t.dt_open >= @CloseFrom AND t.dt_open < DATEADD(DAY,1,@ChkDate)
                                   AND t.out_rub>0
                                  THEN CONVERT(decimal(19,2),t.out_rub) ELSE 0 END),
    vol_dep_bank_1208  = SUM(CASE WHEN t.section_name IN (N'Срочные',N'Срочные ')
                                   AND t.PROD_NAME_res NOT IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                                               N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                                               N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                   AND t.dt_open >= @CloseFrom AND t.dt_open < DATEADD(DAY,1,@ChkDate)
                                   AND t.out_rub>0
                                  THEN CONVERT(decimal(19,2),t.out_rub) ELSE 0 END)
INTO #dep1208
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id = t.cli_id
WHERE t.dt_rep = @ChkDate
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub IS NOT NULL
  AND (t.TSEGMENTNAME IS NULL OR t.TSEGMENTNAME IN (N'Розничный бизнес'))
GROUP BY t.cli_id;
CREATE UNIQUE CLUSTERED INDEX IX_dep1208_cli ON #dep1208(cli_id);

IF OBJECT_ID('tempdb..#ns1208') IS NOT NULL DROP TABLE #ns1208;
SELECT t.cli_id,
       vol_ns_1208 = SUM(CASE WHEN t.out_rub>0 THEN CONVERT(decimal(19,2),t.out_rub) ELSE 0 END)
INTO #ns1208
FROM alm.ALM.balance_rest_all t WITH (NOLOCK)
JOIN #seed31 s ON s.cli_id = t.cli_id
WHERE t.dt_rep = @ChkDate
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub IS NOT NULL
  AND t.section_name IN (N'Накопительный счёт',N'Накопительный счёт ')
  AND (t.TSEGMENTNAME IS NULL OR t.TSEGMENTNAME IN (N'Розничный бизнес'))
GROUP BY t.cli_id;
CREATE UNIQUE CLUSTERED INDEX IX_ns1208_cli ON #ns1208(cli_id);

/* ===================== ПРИНТЫ ===================== */

-- 1) Клиенты, у кого закрываются вклады: кол-во, объём вкладов (31.07), объём НС (31.07)
SELECT
  'SEED: 31.07 только ФУ (все закрытия 01–11.08)' AS label,
  COUNT(*)                                    AS clients_cnt,
  SUM(s.vol_deposits_close31)                 AS deposits_volume_31_close,
  SUM(ISNULL(s.vol_ns31,0))                   AS ns_volume_31
FROM #seed31 s
OPTION (RECOMPILE);

-- 2) 12.08: открыли ТОЛЬКО БАНК — клиенты, объём вкладов, объём НС
SELECT
  'OPEN 01–12.08: только Банк' AS label,
  COUNT(*)                     AS clients_cnt,
  SUM(d.vol_dep_bank_1208)     AS deposits_volume_1208,
  SUM(ISNULL(n.vol_ns_1208,0)) AS ns_volume_1208
FROM #dep1208 d
LEFT JOIN #ns1208 n ON n.cli_id = d.cli_id
WHERE d.dep_open_bank_flag=1 AND d.dep_open_fu_flag=0
OPTION (RECOMPILE);

-- 3) 12.08: открыли ТОЛЬКО ФУ — клиенты, объём вкладов, объём НС
SELECT
  'OPEN 01–12.08: только ФУ'   AS label,
  COUNT(*)                     AS clients_cnt,
  SUM(d.vol_dep_fu_1208)       AS deposits_volume_1208,
  SUM(ISNULL(n.vol_ns_1208,0)) AS ns_volume_1208
FROM #dep1208 d
LEFT JOIN #ns1208 n ON n.cli_id = d.cli_id
WHERE d.dep_open_fu_flag=1 AND d.dep_open_bank_flag=0
OPTION (RECOMPILE);

-- 4) 12.08: открыли ФУ И БАНК — клиенты, общий объём вкладов, объём НС
SELECT
  'OPEN 01–12.08: ФУ и Банк'   AS label,
  COUNT(*)                     AS clients_cnt,
  SUM(d.vol_dep_fu_1208 + d.vol_dep_bank_1208) AS deposits_volume_1208,
  SUM(ISNULL(n.vol_ns_1208,0)) AS ns_volume_1208
FROM #dep1208 d
LEFT JOIN #ns1208 n ON n.cli_id = d.cli_id
WHERE d.dep_open_fu_flag=1 AND d.dep_open_bank_flag=1
OPTION (RECOMPILE);

-- 5) 12.08: НЕ открывали вклады, но НС вырос — клиенты, НС был/стал, изменение
SELECT
  'NO DEPOSIT OPENS 01–12.08, BUT NS INCREASED' AS label,
  COUNT(*)                                      AS clients_cnt,
  SUM(ISNULL(s.vol_ns31,0))                     AS ns_volume_was_31,
  SUM(ISNULL(n.vol_ns_1208,0))                  AS ns_volume_now_1208,
  SUM(ISNULL(n.vol_ns_1208,0) - ISNULL(s.vol_ns31,0)) AS ns_change
FROM #seed31 s
LEFT JOIN #dep1208 d ON d.cli_id = s.cli_id
LEFT JOIN #ns1208 n ON n.cli_id = s.cli_id
WHERE ISNULL(d.dep_open_fu_flag,0)=0 AND ISNULL(d.dep_open_bank_flag,0)=0
  AND ISNULL(n.vol_ns_1208,0) > ISNULL(s.vol_ns31,0)
OPTION (RECOMPILE);

-- 6) Остальные «ушли»: не открывали вклады и НС не вырос (≤ 0) — только количество
SELECT
  'LEFT: no deposit opens & NS not increased' AS label,
  COUNT(*)                                    AS clients_cnt
FROM #seed31 s
LEFT JOIN #dep1208 d ON d.cli_id = s.cli_id
LEFT JOIN #ns1208 n ON n.cli_id = s.cli_id
WHERE ISNULL(d.dep_open_fu_flag,0)=0 AND ISNULL(d.dep_open_bank_flag,0)=0
  AND ISNULL(n.vol_ns_1208,0) <= ISNULL(s.vol_ns31,0)
OPTION (RECOMPILE);
