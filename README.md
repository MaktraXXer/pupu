Отлично, ты сформулировал задачу очень точно.
Итак — у нас три почти идентичных запроса, и нужно внести единый апгрейд, который будет касаться всех трёх:

⸻

🔧 Что нужно исправить в каждом:
	1.	Добавить фильтр по сроку вклада:
term_days BETWEEN 80 AND 100.
👉 Берём term_days (или t.term_days / t.term) из VW_Balance_Rest_All. Если столбец называется иначе — замени на актуальное имя поля, которое отражает срок договора.
	2.	Учесть расширенный набор ставок РК:
	•	AT_THE_END: допускаем 0.165 и 0.163
	•	NOT_AT_THE_END: допускаем 0.162 и 0.160
👉 Это значит, что в фильтрах надо использовать IN (...) вместо точного =.

⸻

✅ Обновлённая версия 1-го скрипта (вклады с РК-ставками)

DECLARE @dt_rep       date        = '2025-10-10';
DECLARE @cur          varchar(3)  = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

-- Расширенный набор РК-ставок
DECLARE @rate_AT_THE_END TABLE (r decimal(9,6));  
INSERT INTO @rate_AT_THE_END VALUES (0.165), (0.163);

DECLARE @rate_NOT_AT_THE_END TABLE (r decimal(9,6));  
INSERT INTO @rate_NOT_AT_THE_END VALUES (0.162), (0.160);

DECLARE @month_start date = DATEFROMPARTS(YEAR(@dt_rep), MONTH(@dt_rep), 1);

;WITH base AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.conv,
        t.term_days
    FROM ALM.ALM.VW_Balance_Rest_All AS t WITH (NOLOCK)
    WHERE
        t.dt_rep = @dt_rep
        AND t.section_name = @section_name
        AND t.block_name   = @block_name
        AND (@od_only = 0 OR t.od_flag = 1)
        AND t.cur = @cur
        AND t.acc_role = @acc_role
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
        AND t.dt_open >= @month_start
        AND t.dt_open <= @dt_rep
        AND t.term_days BETWEEN 80 AND 100
),
filt AS (
    SELECT * 
    FROM base b
    WHERE b.conv = 'AT_THE_END' 
      AND b.rate_con IN (SELECT r FROM @rate_AT_THE_END)

    UNION ALL

    SELECT * 
    FROM base b
    WHERE b.conv <> 'AT_THE_END'
      AND b.rate_con IN (SELECT r FROM @rate_NOT_AT_THE_END)
),
by_con AS (
    SELECT dt_open_d, con_id, MIN(cli_id) AS cli_id, SUM(out_rub) AS out_rub
    FROM filt
    GROUP BY dt_open_d, con_id
),
cal AS (
    SELECT TOP (DATEDIFF(DAY, @month_start, @dt_rep) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @month_start) AS open_date
    FROM master..spt_values
)
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)  AS cnt_deposits_cum,
    COUNT(DISTINCT b.cli_id)  AS cnt_cli_cum,
    SUM(b.out_rub)            AS sum_out_rub_cum,
    CAST(SUM(b.out_rub) / 1e9 AS decimal(18,6)) AS sum_out_rub_cum_bln
FROM cal c
LEFT JOIN by_con b ON b.dt_open_d <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date;


⸻

✅ Обновлённая версия 2-го скрипта (не РК-ставки, но без указанных продуктов)

DECLARE @dt_rep       date = '2025-10-10';
DECLARE @cur          varchar(3) = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit = 1;

-- РК-ставки для исключения
DECLARE @rate_AT_THE_END TABLE (r decimal(9,6));  
INSERT INTO @rate_AT_THE_END VALUES (0.165), (0.163);

DECLARE @rate_NOT_AT_THE_END TABLE (r decimal(9,6));  
INSERT INTO @rate_NOT_AT_THE_END VALUES (0.162), (0.160);

DECLARE @month_start date = DATEFROMPARTS(YEAR(@dt_rep), MONTH(@dt_rep), 1);

WITH base AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.conv,
        t.prod_name_res,
        t.term_days
    FROM ALM.ALM.VW_Balance_Rest_All AS t WITH (NOLOCK)
    WHERE
        t.dt_rep = @dt_rep
        AND t.section_name = @section_name
        AND t.block_name = @block_name
        AND (@od_only = 0 OR t.od_flag = 1)
        AND t.cur = @cur
        AND t.acc_role = @acc_role
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
        AND t.dt_open BETWEEN @month_start AND @dt_rep
        AND t.term_days BETWEEN 80 AND 100
),
filt_nonRK AS (
    SELECT b.*
    FROM base b
    WHERE b.conv = 'AT_THE_END'
      AND (b.rate_con NOT IN (SELECT r FROM @rate_AT_THE_END) OR b.rate_con IS NULL)
      AND b.prod_name_res NOT IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2', N'Надёжный T2',
           N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
      )
    UNION ALL
    SELECT b.*
    FROM base b
    WHERE (b.conv <> 'AT_THE_END' OR b.conv IS NULL)
      AND (b.rate_con NOT IN (SELECT r FROM @rate_NOT_AT_THE_END) OR b.rate_con IS NULL)
      AND b.prod_name_res NOT IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2', N'Надёжный T2',
           N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
      )
),
by_con AS (
    SELECT dt_open_d, con_id, MIN(cli_id) AS cli_id, SUM(out_rub) AS out_rub
    FROM filt_nonRK
    GROUP BY dt_open_d, con_id
),
cal AS (
    SELECT TOP (DATEDIFF(DAY, @month_start, @dt_rep) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @month_start) AS open_date
    FROM master..spt_values
)
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)  AS cnt_deposits_cum,
    COUNT(DISTINCT b.cli_id)  AS cnt_cli_cum,
    SUM(b.out_rub)            AS sum_out_rub_cum,
    CAST(SUM(b.out_rub) / 1e9 AS decimal(18,6)) AS sum_out_rub_cum_bln
FROM cal c
LEFT JOIN by_con b ON b.dt_open_d <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date;


⸻

✅ Обновлённая версия 3-го скрипта (аналог второго, но только по продуктам «Надёжный…» и т. д.)

DECLARE @dt_rep       date = '2025-10-10';
DECLARE @cur          varchar(3) = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit = 1;

-- РК-ставки для исключения
DECLARE @rate_AT_THE_END TABLE (r decimal(9,6));  
INSERT INTO @rate_AT_THE_END VALUES (0.165), (0.163);

DECLARE @rate_NOT_AT_THE_END TABLE (r decimal(9,6));  
INSERT INTO @rate_NOT_AT_THE_END VALUES (0.162), (0.160);

DECLARE @month_start date = DATEFROMPARTS(YEAR(@dt_rep), MONTH(@dt_rep), 1);

WITH base AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.conv,
        t.prod_name_res,
        t.term_days
    FROM ALM.ALM.VW_Balance_Rest_All AS t WITH (NOLOCK)
    WHERE
        t.dt_rep = @dt_rep
        AND t.section_name = @section_name
        AND t.block_name = @block_name
        AND (@od_only = 0 OR t.od_flag = 1)
        AND t.cur = @cur
        AND t.acc_role = @acc_role
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
        AND t.dt_open BETWEEN @month_start AND @dt_rep
        AND t.term_days BETWEEN 80 AND 100
),
filt_nonRK AS (
    SELECT b.*
    FROM base b
    WHERE b.conv = 'AT_THE_END'
      AND (b.rate_con NOT IN (SELECT r FROM @rate_AT_THE_END) OR b.rate_con IS NULL)
      AND b.prod_name_res IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2', N'Надёжный T2',
           N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
      )
    UNION ALL
    SELECT b.*
    FROM base b
    WHERE (b.conv <> 'AT_THE_END' OR b.conv IS NULL)
      AND (b.rate_con NOT IN (SELECT r FROM @rate_NOT_AT_THE_END) OR b.rate_con IS NULL)
      AND b.prod_name_res IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2', N'Надёжный T2',
           N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
      )
),
by_con AS (
    SELECT dt_open_d, con_id, MIN(cli_id) AS cli_id, SUM(out_rub) AS out_rub
    FROM filt_nonRK
    GROUP BY dt_open_d, con_id
),
cal AS (
    SELECT TOP (DATEDIFF(DAY, @month_start, @dt_rep) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @month_start) AS open_date
    FROM master..spt_values
)
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)  AS cnt_deposits_cum,
    COUNT(DISTINCT b.cli_id)  AS cnt_cli_cum,
    SUM(b.out_rub)            AS sum_out_rub_cum,
    CAST(SUM(b.out_rub) / 1e9 AS decimal(18,6)) AS sum_out_rub_cum_bln
FROM cal c
LEFT JOIN by_con b ON b.dt_open_d <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date;


⸻

🧾 Итого
	•	Все три скрипта теперь:
✅ учитывают term_days BETWEEN 80 AND 100,
✅ поддерживают две возможные пары ставок РК (0.165/0.163 и 0.162/0.160),
✅ сохраняют структуру, фильтрацию и накопительный вывод (cnt_deposits_cum, cnt_cli_cum, sum_out_rub_cum, sum_out_rub_cum_bln).

⸻

Хочешь, я сделаю четвёртый объединённый скрипт, который будет выводить три этих выборки сразу, с пометкой категории ('РК', 'Не РК — прочие', 'Не РК — Надёжные') в одном выводе для сравнения?
