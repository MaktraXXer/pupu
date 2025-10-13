да, принял правки. Ниже три скрипта с учётом твоих требований:
	•	bad_cli теперь вычисляется как подмножество ранее полученной когорты (а не глобально) — и исключается из финальной статистики.
	•	В скриптах 1 и 2 добавлен столбец накопительный остаток по НС (на @dt_rep) — ns_out_rub_cum и в млрд.
	•	В скрипте 3 считаем только открытия НС с ненулевым остатком в день открытия и также исключаем bad-клиентов как подмножество когорты.

⸻

Скрипт 1 — «новые вкладные клиенты», кумулятив по вкладам + кумулятив НС-остатков, исключая bad_cli из когорты

/* =========================
   Параметры периода и снимка
   ========================= */
DECLARE @dt_from    date = '2025-10-01';
DECLARE @dt_to      date = '2025-10-10'; -- включительно
DECLARE @dt_rep     date = '2025-10-10'; -- дата снимка баланса
DECLARE @dt_to_next date = DATEADD(day, 1, @dt_to);

DECLARE @cur           varchar(3)   = '810';
DECLARE @section_name  nvarchar(50) = N'Срочные';
DECLARE @block_name    nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role      nvarchar(10) = N'LIAB';
DECLARE @od_only       bit          = 1;

DECLARE @period_end date = CASE WHEN @dt_to <= @dt_rep THEN @dt_to ELSE @dt_rep END;

/* =============== Шаг 0. Чистим временные объекты =============== */
IF OBJECT_ID('tempdb..#new_cli_base') IS NOT NULL DROP TABLE #new_cli_base;
IF OBJECT_ID('tempdb..#bad_cli')      IS NOT NULL DROP TABLE #bad_cli;
IF OBJECT_ID('tempdb..#good_cli')     IS NOT NULL DROP TABLE #good_cli;

/* ================================================================
   Шаг 1. Когорта «новые вкладные клиенты» (без утечки будущего)
   ================================================================ */
WITH base AS (
    SELECT a.CLI_ID, a.CON_ID, a.PROD_NAME, a.DT_OPEN
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    WHERE a.DT_OPEN IS NOT NULL
      AND a.cli_short_name = N'ФЛ'
      AND a.seg_name       = N'Розничный бизнес'
      AND a.cur            = 'RUR'
      -- исключаем тех/эскроу продукты
      AND a.prod_name NOT IN (
            N'Агентские эскроу ФЛ по ставке КС+спред',
            N'Спец. банк.счёт',
            N'Залоговый',
            N'Агентские эскроу ФЛ по ставке КС/2',
            N'Эскроу',
            N'Депозит ФЛ (суррогатный договор для счета Новой Афины)'
      )
),
prior_any AS ( SELECT DISTINCT CLI_ID FROM base WHERE DT_OPEN < @dt_from ),
win_rows  AS ( SELECT * FROM base WHERE DT_OPEN >= @dt_from AND DT_OPEN < @dt_to_next ),
new_clients_now AS (
    SELECT DISTINCT w.CLI_ID
    FROM win_rows w
    WHERE NOT EXISTS (SELECT 1 FROM prior_any p WHERE p.CLI_ID = w.CLI_ID)
),
pure_deposit_clients AS (  -- из «новых» убираем тех, у кого в окне были НС
    SELECT w.CLI_ID
    FROM win_rows w
    JOIN new_clients_now n ON n.CLI_ID = w.CLI_ID
    GROUP BY w.CLI_ID
    HAVING SUM(CASE WHEN w.PROD_NAME IN (N'Накопительный счёт', N'Накопительный счётУльтра', N'Накопительный счёт Ультра') THEN 1 ELSE 0 END) = 0
)
SELECT DISTINCT CLI_ID
INTO #new_cli_base
FROM pure_deposit_clients;

/* ================================================================
   Шаг 1.1. Определяем bad_cli только среди этой когорты (#new_cli_base)
   (на @dt_rep есть активные продукты вне {Срочные, Накопительный счёт, До востребования})
   ================================================================ */
SELECT DISTINCT t.cli_id
INTO #bad_cli
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep
  AND (@od_only = 0 OR t.od_flag = 1)
  AND t.cur          = @cur
  AND t.acc_role     = @acc_role
  AND t.block_name   = @block_name
  AND ISNULL(t.out_rub,0) > 0
  AND t.cli_id IN (SELECT cli_id FROM #new_cli_base)
  AND t.section_name NOT IN (N'Срочные', N'Накопительный счёт', N'До востребования');

/* good-когорта = base − bad */
SELECT cli_id INTO #good_cli
FROM #new_cli_base
WHERE cli_id NOT IN (SELECT cli_id FROM #bad_cli);

/* ==========================================
   Шаг 2. Кумулятив по вкладам + НС-остаток кумулятивно
   ========================================== */
;WITH snap AS (  -- вкладные строки по good-когорте
    SELECT CAST(t.dt_open AS date) AS dt_open_d, t.con_id, t.cli_id, t.out_rub
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE t.dt_rep       = @dt_rep
      AND t.section_name = @section_name
      AND t.block_name   = @block_name
      AND t.acc_role     = @acc_role
      AND (@od_only = 0 OR t.od_flag = 1)
      AND t.cur          = @cur
      AND ISNULL(t.out_rub,0) >= 0
      AND t.dt_open >= @dt_from AND t.dt_open <= @period_end
      AND t.cli_id IN (SELECT cli_id FROM #good_cli)
),
by_con AS (  -- схлопываем дубли по договору
    SELECT dt_open_d, con_id, MIN(cli_id) AS cli_id, SUM(out_rub) AS out_rub
    FROM snap
    GROUP BY dt_open_d, con_id
),
min_open_by_cli AS (  -- первая дата входа клиента в когорту
    SELECT cli_id, MIN(dt_open_d) AS min_open
    FROM by_con
    GROUP BY cli_id
),
ns_bal_by_cli AS (    -- остатки по НС на @dt_rep (>0) по good-когорте
    SELECT t.cli_id, SUM(t.out_rub) AS ns_out_rub
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE t.dt_rep       = @dt_rep
      AND t.block_name   = @block_name
      AND t.acc_role     = @acc_role
      AND (@od_only = 0 OR t.od_flag = 1)
      AND t.cur          = @cur
      AND t.section_name = N'Накопительный счёт'
      AND ISNULL(t.out_rub,0) > 0
      AND t.cli_id IN (SELECT cli_id FROM #good_cli)
    GROUP BY t.cli_id
),
ns_cohort AS ( SELECT m.cli_id, m.min_open, n.ns_out_rub FROM min_open_by_cli m JOIN ns_bal_by_cli n ON n.cli_id = m.cli_id ),
cal AS (
    SELECT TOP (DATEDIFF(DAY, @dt_from, @period_end) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @dt_from) AS open_date
    FROM master..spt_values
)
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)                                AS cnt_deposits_cum,
    COUNT(DISTINCT b.cli_id)                                AS cnt_cli_cum,
    SUM(b.out_rub)                                          AS sum_out_rub_cum,
    CAST(SUM(b.out_rub)/1e9 AS decimal(18,6))               AS sum_out_rub_cum_bln,
    /* кумулятивный остаток НС по когорте (на @dt_rep) */
    SUM(CASE WHEN nc.min_open <= c.open_date THEN nc.ns_out_rub ELSE 0 END)                 AS ns_out_rub_cum,
    CAST(SUM(CASE WHEN nc.min_open <= c.open_date THEN nc.ns_out_rub ELSE 0 END)/1e9 AS decimal(18,6)) AS ns_out_rub_cum_bln
FROM cal c
LEFT JOIN by_con   b  ON b.dt_open_d <= c.open_date
LEFT JOIN ns_cohort nc ON nc.min_open <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date;


⸻

Скрипт 2 — «маркетплейс-вклады»: подмножество скрипта 1 + кумулятив НС-остатков, исключая bad_cli как подмножество когорты

/* Параметры */
DECLARE @dt_from    date = '2025-10-01';
DECLARE @dt_to      date = '2025-10-10';
DECLARE @dt_rep     date = '2025-10-10';
DECLARE @dt_to_next date = DATEADD(day, 1, @dt_to);

DECLARE @cur           varchar(3)   = '810';
DECLARE @section_name  nvarchar(50) = N'Срочные';
DECLARE @block_name    nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role      nvarchar(10) = N'LIAB';
DECLARE @od_only       bit          = 1;
DECLARE @period_end    date = CASE WHEN @dt_to <= @dt_rep THEN @dt_to ELSE @dt_rep END;

/* TMP-таблицы */
IF OBJECT_ID('tempdb..#hui_base')  IS NOT NULL DROP TABLE #hui_base;
IF OBJECT_ID('tempdb..#bad_cli')   IS NOT NULL DROP TABLE #bad_cli;
IF OBJECT_ID('tempdb..#good_cli')  IS NOT NULL DROP TABLE #good_cli;

/* Когорта «новые вкладные» → «первый в окне — маркетплейс» */
WITH base AS ( 
    SELECT a.CLI_ID, a.CON_ID, a.PROD_NAME, a.DT_OPEN
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    WHERE a.DT_OPEN IS NOT NULL
      AND a.cli_short_name = N'ФЛ'
      AND a.seg_name       = N'Розничный бизнес'
      AND a.cur            = 'RUR'
      AND a.prod_name NOT IN ( 
            N'Агентские эскроу ФЛ по ставке КС+спред',
            N'Спец. банк.счёт',
            N'Залоговый',
            N'Агентские эскроу ФЛ по ставке КС/2',
            N'Эскроу',
            N'Депозит ФЛ (суррогатный договор для счета Новой Афины)'
      )
),
prior_any AS ( SELECT DISTINCT CLI_ID FROM base WHERE DT_OPEN < @dt_from ),
win_rows  AS ( SELECT * FROM base WHERE DT_OPEN >= @dt_from AND DT_OPEN < @dt_to_next ),
new_clients_now AS (
    SELECT DISTINCT w.CLI_ID FROM win_rows w
    WHERE NOT EXISTS (SELECT 1 FROM prior_any p WHERE p.CLI_ID = w.CLI_ID)
),
pure_deposit_clients AS (
    SELECT w.CLI_ID
    FROM win_rows w JOIN new_clients_now n ON n.CLI_ID = w.CLI_ID
    GROUP BY w.CLI_ID
    HAVING SUM(CASE WHEN w.PROD_NAME IN (N'Накопительный счёт', N'Накопительный счётУльтра', N'Накопительный счёт Ультра') THEN 1 ELSE 0 END) = 0
),
first_in_window AS (
    SELECT w.*, ROW_NUMBER() OVER (PARTITION BY w.CLI_ID ORDER BY w.DT_OPEN, w.CON_ID) AS rn
    FROM win_rows w JOIN pure_deposit_clients p ON p.CLI_ID = w.CLI_ID
),
mkt_first AS (
    SELECT CLI_ID
    FROM first_in_window
    WHERE rn = 1
      AND PROD_NAME IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт',
           N'Надёжный Т2', N'Надёжный T2',
           N'Надёжный Мегафон', N'Надёжный процент',
           N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
      )
)
SELECT DISTINCT CLI_ID INTO #hui_base FROM mkt_first;

/* bad_cli только внутри когорты (#hui_base) */
SELECT DISTINCT t.cli_id
INTO #bad_cli
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep
  AND (@od_only = 0 OR t.od_flag = 1)
  AND t.cur          = @cur
  AND t.acc_role     = @acc_role
  AND t.block_name   = @block_name
  AND ISNULL(t.out_rub,0) > 0
  AND t.cli_id IN (SELECT cli_id FROM #hui_base)
  AND t.section_name NOT IN (N'Срочные', N'Накопительный счёт', N'До востребования');

/* good-когорта */
SELECT cli_id INTO #good_cli
FROM #hui_base
WHERE cli_id NOT IN (SELECT cli_id FROM #bad_cli);

/* Снимок и кумулятивы + НС-остатки */
;WITH snap AS (
    SELECT CAST(t.dt_open AS date) AS dt_open_d, t.con_id, t.cli_id, t.out_rub
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE t.dt_rep       = @dt_rep
      AND t.section_name = @section_name
      AND t.block_name   = @block_name
      AND t.acc_role     = @acc_role
      AND (@od_only = 0 OR t.od_flag = 1)
      AND t.cur          = @cur
      AND ISNULL(t.out_rub,0) >= 0
      AND t.dt_open >= @dt_from AND t.dt_open <= @period_end
      AND t.cli_id IN (SELECT cli_id FROM #good_cli)
),
by_con AS (
    SELECT dt_open_d, con_id, MIN(cli_id) AS cli_id, SUM(out_rub) AS out_rub
    FROM snap GROUP BY dt_open_d, con_id
),
min_open_by_cli AS ( SELECT cli_id, MIN(dt_open_d) AS min_open FROM by_con GROUP BY cli_id ),
ns_bal_by_cli  AS (
    SELECT t.cli_id, SUM(t.out_rub) AS ns_out_rub
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE t.dt_rep       = @dt_rep
      AND t.block_name   = @block_name
      AND t.acc_role     = @acc_role
      AND (@od_only = 0 OR t.od_flag = 1)
      AND t.cur          = @cur
      AND t.section_name = N'Накопительный счёт'
      AND ISNULL(t.out_rub,0) > 0
      AND t.cli_id IN (SELECT cli_id FROM #good_cli)
    GROUP BY t.cli_id
),
ns_cohort AS ( SELECT m.cli_id, m.min_open, n.ns_out_rub FROM min_open_by_cli m JOIN ns_bal_by_cli n ON n.cli_id = m.cli_id ),
cal AS (
    SELECT TOP (DATEDIFF(DAY, @dt_from, @period_end) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @dt_from) AS open_date
    FROM master..spt_values
)
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)                              AS cnt_deposits_cum,
    COUNT(DISTINCT b.cli_id)                              AS cnt_cli_cum,
    SUM(b.out_rub)                                        AS sum_out_rub_cum,
    CAST(SUM(b.out_rub)/1e9 AS decimal(18,6))             AS sum_out_rub_cum_bln,
    /* кумулятив НС-остатков (на @dt_rep) */
    SUM(CASE WHEN nc.min_open <= c.open_date THEN nc.ns_out_rub ELSE 0 END)                 AS ns_out_rub_cum,
    CAST(SUM(CASE WHEN nc.min_open <= c.open_date THEN nc.ns_out_rub ELSE 0 END)/1e9 AS decimal(18,6)) AS ns_out_rub_cum_bln
FROM cal c
LEFT JOIN by_con   b  ON b.dt_open_d <= c.open_date
LEFT JOIN ns_cohort nc ON nc.min_open <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date;


⸻

Скрипт 3 — «новые НС-клиенты» (инкремент на дату открытия out_rub>0), исключая bad_cli как подмножество когорты

/* Параметры */
DECLARE @dt_from    date = '2025-10-01';
DECLARE @dt_to      date = '2025-10-10';
DECLARE @dt_rep     date = '2025-10-10';
DECLARE @dt_to_next date = DATEADD(day, 1, @dt_to);

DECLARE @cur           varchar(3)   = '810';
DECLARE @block_name    nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role      nvarchar(10) = N'LIAB';
DECLARE @od_only       bit          = 1;

DECLARE @period_end date = CASE WHEN @dt_to <= @dt_rep THEN @dt_to ELSE @dt_rep END;

/* TMP-таблицы */
IF OBJECT_ID('tempdb..#new_ns_cli_base') IS NOT NULL DROP TABLE #new_ns_cli_base;
IF OBJECT_ID('tempdb..#bad_cli')         IS NOT NULL DROP TABLE #bad_cli;
IF OBJECT_ID('tempdb..#good_cli')        IS NOT NULL DROP TABLE #good_cli;

/* Когорта «новые с НС в окне», без утечки будущего */
WITH base AS (
    SELECT a.CLI_ID, a.CON_ID, a.PROD_NAME, a.DT_OPEN
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    WHERE a.DT_OPEN IS NOT NULL
      AND a.cli_short_name = N'ФЛ'
      AND a.seg_name       = N'Розничный бизнес'
      AND a.cur            = 'RUR'
      AND a.prod_name NOT IN (
            N'Агентские эскроу ФЛ по ставке КС+спред',
            N'Спец. банк.счёт',
            N'Залоговый',
            N'Агентские эскроу ФЛ по ставке КС/2',
            N'Эскроу',
            N'Депозит ФЛ (суррогатный договор для счета Новой Афины)'
      )
),
prior_any AS ( SELECT DISTINCT CLI_ID FROM base WHERE DT_OPEN < @dt_from ),
win_rows  AS ( SELECT * FROM base WHERE DT_OPEN >= @dt_from AND DT_OPEN < @dt_to_next ),
new_clients_now AS (
    SELECT DISTINCT w.CLI_ID
    FROM win_rows w
    WHERE NOT EXISTS (SELECT 1 FROM prior_any p WHERE p.CLI_ID = w.CLI_ID)
),
ns_clients AS (
    SELECT DISTINCT w.CLI_ID
    FROM win_rows w JOIN new_clients_now n ON n.CLI_ID = w.CLI_ID
    WHERE w.PROD_NAME IN (N'Накопительный счёт', N'Накопительный счётУльтра', N'Накопительный счёт Ультра')
)
SELECT DISTINCT CLI_ID INTO #new_ns_cli_base FROM ns_clients;

/* bad_cli только внутри когорты (#new_ns_cli_base), на @dt_rep есть прочие продукты */
SELECT DISTINCT t.cli_id
INTO #bad_cli
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep
  AND (@od_only = 0 OR t.od_flag = 1)
  AND t.cur          = @cur
  AND t.acc_role     = @acc_role
  AND t.block_name   = @block_name
  AND ISNULL(t.out_rub,0) > 0
  AND t.cli_id IN (SELECT cli_id FROM #new_ns_cli_base)
  AND t.section_name NOT IN (N'Срочные', N'Накопительный счёт', N'До востребования');

/* good-когорта */
SELECT cli_id INTO #good_cli
FROM #new_ns_cli_base
WHERE cli_id NOT IN (SELECT cli_id FROM #bad_cli);

/* Инкременты на дату открытия (только out_rub>0), затем кумулятив */
;WITH opened_ns AS (
    SELECT CAST(t.dt_open AS date) AS dt_open_d, t.con_id, t.cli_id, t.out_rub
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE t.dt_rep BETWEEN @dt_from AND @period_end
      AND t.dt_rep   = CAST(t.dt_open AS date)                   -- ровно день открытия
      AND (@od_only = 0 OR t.od_flag = 1)
      AND t.cur          = @cur
      AND t.block_name   = @block_name
      AND t.acc_role     = @acc_role
      AND ISNULL(t.out_rub,0) > 0                                -- <— только ненулевой остаток на дату открытия
      AND t.section_name = N'Накопительный счёт'
      AND t.cli_id IN (SELECT cli_id FROM #good_cli)             -- <— исключаем bad-клиентов
),
by_con AS (
    SELECT dt_open_d, con_id, MIN(cli_id) AS cli_id, SUM(out_rub) AS out_rub
    FROM opened_ns
    GROUP BY dt_open_d, con_id
),
daily AS (
    SELECT dt_open_d,
           COUNT(DISTINCT con_id) AS cnt_ns_accounts_day,
           COUNT(DISTINCT cli_id) AS cnt_ns_clients_day,
           SUM(out_rub)           AS sum_ns_out_rub_day
    FROM by_con
    GROUP BY dt_open_d
),
cal AS (
    SELECT TOP (DATEDIFF(DAY, @dt_from, @period_end) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @dt_from) AS open_date
    FROM master..spt_values
),
daily_filled AS (
    SELECT c.open_date,
           ISNULL(d.cnt_ns_accounts_day,0) AS cnt_ns_accounts_day,
           ISNULL(d.cnt_ns_clients_day,0)  AS cnt_ns_clients_day,
           ISNULL(d.sum_ns_out_rub_day,0)  AS sum_ns_out_rub_day
    FROM cal c
    LEFT JOIN daily d ON d.dt_open_d = c.open_date
)
SELECT
    open_date,
    SUM(cnt_ns_accounts_day) OVER (ORDER BY open_date ROWS UNBOUNDED PRECEDING) AS cnt_ns_accounts_cum,
    SUM(cnt_ns_clients_day)  OVER (ORDER BY open_date ROWS UNBOUNDED PRECEDING) AS cnt_ns_clients_cum,
    SUM(sum_ns_out_rub_day)  OVER (ORDER BY open_date ROWS UNBOUNDED PRECEDING) AS sum_ns_out_rub_cum,
    CAST(SUM(sum_ns_out_rub_day) OVER (ORDER BY open_date ROWS UNBOUNDED PRECEDING)/1e9 AS decimal(18,6)) AS sum_ns_out_rub_cum_bln
FROM daily_filled
ORDER BY open_date;

Если нужно — добавлю диагностический селект, что именно делает клиента bad (по конкретным section_name/prod_name_res) уже внутри когорты.
