/* === ПАРАМЕТРЫ (как в скрипте 1) === */
DECLARE @dt_rep       date         = '2025-10-10';
DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

-- РК-ставки (как в скрипте 1) — только для построения множества rk_ids
DECLARE @rate_AT_THE_END TABLE (r decimal(9,6));  INSERT INTO @rate_AT_THE_END VALUES (0.165),(0.163);
DECLARE @rate_NOT_AT_THE_END TABLE (r decimal(9,6));  INSERT INTO @rate_NOT_AT_THE_END VALUES (0.162),(0.160);

DECLARE @month_start date = DATEFROMPARTS(YEAR(@dt_rep), MONTH(@dt_rep), 1);

/* === Общая база за окно с начала месяца до @dt_rep === */
WITH base AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.conv,
        t.prod_name_res
    FROM ALM.ALM.VW_Balance_Rest_All AS t WITH (NOLOCK)
    WHERE
        t.dt_rep       = @dt_rep
        AND t.section_name = @section_name
        AND t.block_name   = @block_name
        AND (@od_only = 0 OR t.od_flag = 1)
        AND t.cur          = @cur
        AND t.acc_role     = @acc_role
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
        AND t.dt_open BETWEEN @month_start AND @dt_rep
),
/* === РК-множество из СКРИПТА 1 (строго те же правила!) === */
rk_filt AS (
    SELECT * FROM base b
    WHERE b.conv = 'AT_THE_END'
      AND b.rate_con IN (SELECT r FROM @rate_AT_THE_END)
    UNION ALL
    SELECT * FROM base b
    WHERE (b.conv <> 'AT_THE_END')   -- NULL не попадёт в РК
      AND b.rate_con IN (SELECT r FROM @rate_NOT_AT_THE_END)
),
rk_ids AS (    -- distinct договоры РК
    SELECT DISTINCT con_id FROM rk_filt
),
/* === НЕ-РК = base \ rk_ids (никаких условий по ставкам здесь!) === */
nonrk_base AS (
    SELECT b.*
    FROM base b
    WHERE NOT EXISTS (SELECT 1 FROM rk_ids r WHERE r.con_id = b.con_id)
),
/* === Скрипт 2: из НЕ-РК исключаем набор продуктов === */
nonrk_filtered AS (
    SELECT *
    FROM nonrk_base
    WHERE prod_name_res NOT IN (
        N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
        N'Надёжный промо', N'Надёжный старт',
        N'Надёжный Т2', N'Надёжный T2',
        N'Надёжный Мегафон', N'Надёжный процент',
        N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
    )
),
by_con AS (  -- схлопываем по договору (на дату открытия)
    SELECT
        dt_open_d,
        con_id,
        MIN(cli_id)  AS cli_id,
        SUM(out_rub) AS out_rub
    FROM nonrk_filtered
    GROUP BY dt_open_d, con_id
),
cal AS (     -- календарь для накопительного итога
    SELECT TOP (DATEDIFF(DAY, @month_start, @dt_rep) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @month_start) AS open_date
    FROM master..spt_values
)
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)                    AS cnt_deposits_cum,
    COUNT(DISTINCT b.cli_id)                    AS cnt_cli_cum,
    SUM(b.out_rub)                              AS sum_out_rub_cum,
    CAST(SUM(b.out_rub) / 1e9 AS decimal(18,6)) AS sum_out_rub_cum_bln
FROM cal c
LEFT JOIN by_con b
       ON b.dt_open_d <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date;
