/* === ПАРАМЕТРЫ === */
DECLARE @dt_rep       date        = '2025-10-10';          -- дата снимка баланса
DECLARE @cur          varchar(3)  = '810';                 -- валюта (810 = RUB)
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;                    -- учитывать только OD (=1)

-- Эталонные РК-ставки под конвенции:
DECLARE @rate_AT_THE_END     decimal(9,6) = 0.165;         -- conv = 'AT_THE_END'
DECLARE @rate_NOT_AT_THE_END decimal(9,6) = 0.162;         -- conv <> 'AT_THE_END'

DECLARE @month_start date = DATEFROMPARTS(YEAR(@dt_rep), MONTH(@dt_rep), 1);

/* === ОСНОВНОЙ ЗАПРОС === */
WITH base AS (
   SELECT
       CAST(t.dt_open AS date) AS dt_open_d,
       t.con_id,
       t.cli_id,
       t.out_rub,
       t.rate_con,
       t.conv,
       t.prod_name
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
/* НЕ по ставке РК + исключаем перечисленные продукты */
filt_nonRK AS (
   SELECT b.*
   FROM base b
   WHERE b.conv = 'AT_THE_END'
     AND (b.rate_con <> @rate_AT_THE_END OR b.rate_con IS NULL)
     AND b.PROD_NAME NOT IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт',
           N'Надёжный Т2', N'Надёжный T2',
           N'Надёжный Мегафон', N'Надёжный процент',
           N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
     )

   UNION ALL

   SELECT b.*
   FROM base b
   WHERE b.conv <> 'AT_THE_END'
     AND (b.rate_con <> @rate_NOT_AT_THE_END OR b.rate_con IS NULL)
     AND b.PROD_NAME NOT IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт',
           N'Надёжный Т2', N'Надёжный T2',
           N'Надёжный Мегафон', N'Надёжный процент',
           N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
     )
),
by_con AS (
   SELECT
       dt_open_d,
       con_id,
       MIN(cli_id)  AS cli_id,
       SUM(out_rub) AS out_rub
   FROM filt_nonRK
   GROUP BY dt_open_d, con_id
)
SELECT
   dt_open_d                                   AS [open_date],
   COUNT(*)                                    AS [cnt_deposits],          -- число вкладов
   COUNT(DISTINCT cli_id)                      AS [cnt_cli],               -- уникальных клиентов
   SUM(out_rub)                                AS [sum_out_rub],           -- сумма, руб
   CAST(SUM(out_rub) / 1e9 AS decimal(18,6))   AS [sum_out_rub_bln]        -- сумма, млрд
FROM by_con
GROUP BY dt_open_d
ORDER BY [open_date];
