/* === ПАРАМЕТРЫ === */
DECLARE @dt_rep       date         = '2025-10-10';          -- дата снимка баланса
DECLARE @cur          varchar(3)   = '810';                 -- валюта (810 = RUB)
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;                     -- учитывать только OD (=1)

-- Эталонные РК-ставки под конвенции:
DECLARE @rate_AT_THE_END     decimal(9,6) = 0.165;          -- conv = 'AT_THE_END'
DECLARE @rate_NOT_AT_THE_END decimal(9,6) = 0.162;          -- conv <> 'AT_THE_END'

/* === ГРАНИЦЫ ОКНА (автоматически от начала месяца до @dt_rep) === */
DECLARE @month_start date = DATEFROMPARTS(YEAR(@dt_rep), MONTH(@dt_rep), 1);

/* === ОСНОВНОЙ ЗАПРОС (НЕ по ставке РК + исключаем заданные продукты по prod_name_res) === */
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
        AND t.dt_open >= @month_start
        AND t.dt_open <= @dt_rep
),
filt_nonRK AS (
    /* conv = 'AT_THE_END' и ставка НЕ равна эталонной (или NULL) */
    SELECT b.*
    FROM base b
    WHERE b.conv = 'AT_THE_END'
      AND (b.rate_con <> @rate_AT_THE_END OR b.rate_con IS NULL)
      AND b.prod_name_res NOT IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт',
           N'Надёжный Т2', N'Надёжный T2',
           N'Надёжный Мегафон', N'Надёжный процент',
           N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
      )

    UNION ALL

    /* все остальные конвенции (включая NULL), ставка НЕ равна эталонной (или NULL) */
    SELECT b.*
    FROM base b
    WHERE (b.conv <> 'AT_THE_END' OR b.conv IS NULL)
      AND (b.rate_con <> @rate_NOT_AT_THE_END OR b.rate_con IS NULL)
      AND b.prod_name_res NOT IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт',
           N'Надёжный Т2', N'Надёжный T2',
           N'Надёжный Мегафон', N'Надёжный процент',
           N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
      )
),
by_con AS (  -- схлопываем возможные дубли по договору
    SELECT
        dt_open_d,
        con_id,
        MIN(cli_id)  AS cli_id,
        SUM(out_rub) AS out_rub
    FROM filt_nonRK
    GROUP BY dt_open_d, con_id
),
cal AS (     -- календарь с начала месяца до @dt_rep
    SELECT TOP (DATEDIFF(DAY, @month_start, @dt_rep) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @month_start) AS open_date
    FROM master..spt_values
)
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)                            AS cnt_deposits_cum,     -- кумулятив по договорам
    COUNT(DISTINCT b.cli_id)                            AS cnt_cli_cum,          -- кумулятив по клиентам
    SUM(b.out_rub)                                      AS sum_out_rub_cum,      -- кумулятив сумма, руб
    CAST(SUM(b.out_rub) / 1e9 AS decimal(18,6))         AS sum_out_rub_cum_bln   -- кумулятив сумма, млрд
FROM cal c
LEFT JOIN by_con b
       ON b.dt_open_d <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date;
