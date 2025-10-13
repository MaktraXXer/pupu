/* =========================
   Параметры периода и снимка
   ========================= */
DECLARE @dt_from    date = '2025-10-01';
DECLARE @dt_to      date = '2025-10-10'; -- включительно
DECLARE @dt_rep     date = '2025-10-10'; -- дата максимального снапшота, если нужна
DECLARE @dt_to_next date = DATEADD(day, 1, @dt_to);

DECLARE @cur           varchar(3)   = '810';
DECLARE @block_name    nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role      nvarchar(10) = N'LIAB';
DECLARE @od_only       bit          = 1;

/* Период для календаря: от @dt_from до min(@dt_to, @dt_rep) */
DECLARE @period_end date = CASE WHEN @dt_to <= @dt_rep THEN @dt_to ELSE @dt_rep END;

/* =============== Шаг 0. Чистим временные объекты =============== */
IF OBJECT_ID('tempdb..#new_ns_cli') IS NOT NULL DROP TABLE #new_ns_cli;

/* ================================================================
   Шаг 1. Новые клиенты в окне, у которых В ОКНЕ есть Накопительный счёт
   (без утечки будущего — новизна проверяется только до @dt_from)
   ================================================================ */
WITH base AS (
    SELECT a.CLI_ID, a.CON_ID, a.PROD_NAME, a.DT_OPEN
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    WHERE a.DT_OPEN IS NOT NULL
      AND a.cli_short_name = N'ФЛ'
      AND a.seg_name       = N'Розничный бизнес'
      AND a.cur            = 'RUR'
      -- исключаем тех/эскроу-продукты, НС НЕ исключаем
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
ns_clients AS (  -- у кого в окне есть хотя бы один НС
    SELECT DISTINCT w.CLI_ID
    FROM win_rows w
    JOIN new_clients_now n ON n.CLI_ID = w.CLI_ID
    WHERE w.PROD_NAME IN (N'Накопительный счёт', N'Накопительный счётУльтра', N'Накопительный счёт Ультра')
)
SELECT DISTINCT CLI_ID
INTO #new_ns_cli
FROM ns_clients;

/* ================================================================
   Шаг 2. Инкременты «на дату открытия» по снапшотам VW_Balance_Rest_All:
   считаем только НС, только по #new_ns_cli, и ТОЛЬКО если на дате открытия out_rub > 0
   (используем строки где dt_rep = CAST(dt_open AS date))
   ================================================================ */
;WITH opened_ns AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE t.dt_rep BETWEEN @dt_from AND @period_end
      AND t.dt_rep   = CAST(t.dt_open AS date)                   -- берём срез ровно на дату открытия
      AND (@od_only = 0 OR t.od_flag = 1)
      AND t.cur          = @cur
      AND t.block_name   = @block_name
      AND t.acc_role     = @acc_role
      AND t.out_rub IS NOT NULL
      AND t.out_rub > 0                                        -- <— только ненулевой остаток
      AND t.prod_name_res IN (N'Накопительный счёт', N'Накопительный счётУльтра', N'Накопительный счёт Ультра')
      AND t.cli_id IN (SELECT cli_id FROM #new_ns_cli)
),
by_con AS (  -- на всякий: схлопываем дубли по договору на дату
    SELECT
        dt_open_d,
        con_id,
        MIN(cli_id)  AS cli_id,
        SUM(out_rub) AS out_rub
    FROM opened_ns
    GROUP BY dt_open_d, con_id
),
daily AS (  -- дневные инкременты
    SELECT
        dt_open_d,
        COUNT(DISTINCT con_id) AS cnt_ns_accounts_day,
        COUNT(DISTINCT cli_id) AS cnt_ns_clients_day,
        SUM(out_rub)           AS sum_ns_out_rub_day
    FROM by_con
    GROUP BY dt_open_d
),
cal AS (     -- календарь по дням периода
    SELECT TOP (DATEDIFF(DAY, @dt_from, @period_end) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @dt_from) AS open_date
    FROM master..spt_values
),
daily_filled AS ( -- заполним пропуски нулями
    SELECT
        c.open_date,
        ISNULL(d.cnt_ns_accounts_day, 0) AS cnt_ns_accounts_day,
        ISNULL(d.cnt_ns_clients_day, 0)  AS cnt_ns_clients_day,
        ISNULL(d.sum_ns_out_rub_day, 0)  AS sum_ns_out_rub_day
    FROM cal c
    LEFT JOIN daily d ON d.dt_open_d = c.open_date
)
SELECT
    open_date,
    /* кумулятив */
    SUM(cnt_ns_accounts_day) OVER (ORDER BY open_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)           AS cnt_ns_accounts_cum,
    SUM(cnt_ns_clients_day)  OVER (ORDER BY open_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)           AS cnt_ns_clients_cum,
    SUM(sum_ns_out_rub_day)  OVER (ORDER BY open_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)           AS sum_ns_out_rub_cum,
    CAST(
        SUM(sum_ns_out_rub_day) OVER (ORDER BY open_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) / 1e9
        AS decimal(18,6)
    ) AS sum_ns_out_rub_cum_bln,
    /* для удобства — дневные инкременты рядом */
    cnt_ns_accounts_day,
    cnt_ns_clients_day,
    sum_ns_out_rub_day,
    CAST(sum_ns_out_rub_day / 1e9 AS decimal(18,6)) AS sum_ns_out_rub_day_bln
FROM daily_filled
ORDER BY open_date;
