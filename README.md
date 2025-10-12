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
-- Фильтруем по ставке и конвенции (две ветки, быстрее чем OR)
filt AS (
    SELECT * FROM base WHERE conv = 'AT_THE_END'  AND rate_con = @rate_AT_THE_END
    UNION ALL
    SELECT * FROM base WHERE conv <> 'AT_THE_END' AND rate_con = @rate_NOT_AT_THE_END
),
-- Схлопываем по con_id (если дубли)
by_con AS (
    SELECT
        dt_open_d,
        con_id,
        MIN(cli_id) AS cli_id,
        SUM(out_rub) AS out_rub
    FROM filt
    GROUP BY dt_open_d, con_id
),
-- Агрегируем дневные значения (вклады, клиенты, суммы)
daily AS (
    SELECT
        dt_open_d AS open_date,
        COUNT(*) AS cnt_deposits,
        COUNT(DISTINCT cli_id) AS cnt_cli_day,
        SUM(out_rub) AS sum_out_rub,
        CAST(SUM(out_rub) / 1e9 AS decimal(18,6)) AS sum_out_rub_bln
    FROM by_con
    GROUP BY dt_open_d
),
-- Находим накопительный итог по уникальным клиентам
cumulative_clients AS (
    SELECT
        d1.open_date,
        COUNT(DISTINCT b.cli_id) AS cnt_cli_cum
    FROM daily d1
    JOIN by_con b
      ON b.dt_open_d <= d1.open_date
    GROUP BY d1.open_date
)
-- Финальный вывод
SELECT
    d.open_date,
    d.cnt_deposits,
    d.cnt_cli_day,                      -- новые клиенты за день (уникальные за дату)
    c.cnt_cli_cum,                      -- накопительный итог уникальных клиентов
    d.sum_out_rub,
    d.sum_out_rub_bln
FROM daily d
JOIN cumulative_clients c ON c.open_date = d.open_date
ORDER BY d.open_date;
