DECLARE @dt_rep       date        = '2025-10-10';          -- дата снимка
DECLARE @cur          varchar(3)  = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;                    -- учитывать только OD (=1)

-- Значения ставок под разные конвенции (под условия РК)
DECLARE @rate_AT_THE_END    decimal(9,6) = 0.165;          -- conv = 'AT_THE_END'
DECLARE @rate_NOT_AT_THE_END decimal(9,6) = 0.162;         -- conv <> 'AT_THE_END'

-- Автоопределение начала месяца
DECLARE @month_start date = DATEFROMPARTS(YEAR(@dt_rep), MONTH(@dt_rep), 1);

;WITH base AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.conv
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
        AND t.dt_open >= @month_start
        AND t.dt_open <= @dt_rep
),
-- Фильтрация по ставке и конвенции
filt AS (
    SELECT * FROM base WHERE conv = 'AT_THE_END'  AND rate_con = @rate_AT_THE_END
    UNION ALL
    SELECT * FROM base WHERE conv <> 'AT_THE_END' AND rate_con = @rate_NOT_AT_THE_END
),
-- Убираем возможные дубли con_id
by_con AS (
    SELECT
        dt_open_d,
        con_id,
        MIN(cli_id) AS cli_id,
        SUM(out_rub) AS out_rub
    FROM filt
    GROUP BY dt_open_d, con_id
),
-- Даты, на которые будем считать накопительный итог
cal AS (
    SELECT TOP (DATEDIFF(DAY, @month_start, @dt_rep) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @month_start) AS open_date
    FROM master..spt_values
)
-- Финальный расчёт накопительных итогов
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)      AS cnt_deposits_cum,   -- количество вкладов с начала месяца
    COUNT(DISTINCT b.cli_id)      AS cnt_cli_cum,        -- уникальные клиенты с начала месяца
    SUM(b.out_rub)                AS sum_out_rub_cum,    -- сумма вкладов с начала месяца
    CAST(SUM(b.out_rub) / 1e9 AS decimal(18,6)) AS sum_out_rub_cum_bln
FROM cal c
LEFT JOIN by_con b
       ON b.dt_open_d <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date;
