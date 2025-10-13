/* =========================
   Параметры периода и снимка
   ========================= */
DECLARE @dt_from    date = '2025-10-01';
DECLARE @dt_to      date = '2025-10-10'; -- включительно
DECLARE @dt_rep     date = '2025-10-10'; -- дата снимка баланса
DECLARE @dt_to_next date = DATEADD(day, 1, @dt_to);

DECLARE @cur           varchar(3)   = '810';
DECLARE @section_name  nvarchar(50) = N'Срочные';          -- для НС можно не использовать, ниже фильтруем по prod_name_res
DECLARE @block_name    nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role      nvarchar(10) = N'LIAB';
DECLARE @od_only       bit          = 1;

/* Период для календаря: от @dt_from до min(@dt_to, @dt_rep) */
DECLARE @period_end date = CASE WHEN @dt_to <= @dt_rep THEN @dt_to ELSE @dt_rep END;

/* =============== Шаг 0. Чистим временные объекты =============== */
IF OBJECT_ID('tempdb..#new_ns_cli') IS NOT NULL DROP TABLE #new_ns_cli;

/* ================================================================
   Шаг 1. Новые клиенты в окне, у которых В ОКНЕ есть Накопительный счёт
   (без утечки будущего)
   ================================================================ */
WITH base AS (
    SELECT a.CLI_ID, a.CON_ID, a.PROD_NAME, a.DT_OPEN
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    WHERE a.DT_OPEN IS NOT NULL
      AND a.cli_short_name = N'ФЛ'
      AND a.seg_name       = N'Розничный бизнес'
      AND a.cur            = 'RUR'
      -- исключаем тех/эскроу-продукты (НC НЕ исключаем!)
      AND a.prod_name NOT IN (
            N'Агентские эскроу ФЛ по ставке КС+спред',
            N'Спец. банк.счёт',
            N'Залоговый',
            N'Агентские эскроу ФЛ по ставке КС/2',
            N'Эскроу',
            N'Депозит ФЛ (суррогатный договор для счета Новой Афины)'
      )
),
prior_any AS (   -- любая история раньше окна?
    SELECT DISTINCT CLI_ID
    FROM base
    WHERE DT_OPEN < @dt_from
),
win_rows AS (    -- строки в самом окне
    SELECT *
    FROM base
    WHERE DT_OPEN >= @dt_from
      AND DT_OPEN <  @dt_to_next
),
new_clients_now AS (  -- новые на момент окна
    SELECT DISTINCT w.CLI_ID
    FROM win_rows w
    WHERE NOT EXISTS (SELECT 1 FROM prior_any p WHERE p.CLI_ID = w.CLI_ID)
),
-- клиенты, у кого в ОКНЕ есть хотя бы один НС
ns_clients AS (
    SELECT DISTINCT w.CLI_ID
    FROM win_rows w
    JOIN new_clients_now n ON n.CLI_ID = w.CLI_ID
    WHERE w.PROD_NAME IN (N'Накопительный счёт', N'Накопительный счётУльтра', N'Накопительный счёт Ультра')
)
SELECT DISTINCT CLI_ID
INTO #new_ns_cli
FROM ns_clients;

/* ==========================================
   Шаг 2. Кумулятив по снимку баланса @dt_rep
   Только по #new_ns_cli и только НС-продукты
   ========================================== */
;WITH snap AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.prod_name_res
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE t.dt_rep       = @dt_rep
      -- секцию можно не фиксировать, фильтруем по prod_name_res:
      -- AND t.section_name = @section_name
      AND t.block_name   = @block_name
      AND t.acc_role     = @acc_role
      AND (@od_only = 0 OR t.od_flag = 1)
      AND t.cur          = @cur
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0
      AND t.dt_open >= @dt_from
      AND t.dt_open <= @period_end
      AND t.cli_id IN (SELECT cli_id FROM #new_ns_cli)
      AND t.prod_name_res IN (N'Накопительный счёт', N'Накопительный счётУльтра', N'Накопительный счёт Ультра')
),
by_con AS (  -- схлопываем дубли по одному договору
    SELECT
        dt_open_d,
        con_id,
        MIN(cli_id)  AS cli_id,
        SUM(out_rub) AS out_rub
    FROM snap
    GROUP BY dt_open_d, con_id
),
cal AS (     -- календарь по дням периода
    SELECT TOP (DATEDIFF(DAY, @dt_from, @period_end) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @dt_from) AS open_date
    FROM master..spt_values
)
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)                    AS cnt_ns_accounts_cum,  -- кумулятив по НС-счетам
    COUNT(DISTINCT b.cli_id)                    AS cnt_ns_clients_cum,   -- кумулятив по клиентам НС
    SUM(b.out_rub)                              AS sum_ns_out_rub_cum,   -- кумулятив сумма, руб
    CAST(SUM(b.out_rub) / 1e9 AS decimal(18,6)) AS sum_ns_out_rub_cum_bln
FROM cal c
LEFT JOIN by_con b
       ON b.dt_open_d <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date;
