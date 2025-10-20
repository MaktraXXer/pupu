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
        t.termdays
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
        AND t.termdays BETWEEN 80 AND 100
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
 
Вот мой скрипт где я выделял накопительным итогом выделял обьемы по ставке РК – там прописано какая именно ставка
 
 
Ниже второй скрипт где я беру и исключаю сделки которые попали в выгрузку 1! Важно именно так исключать! – и вывожу динамику с накопительным итогом но только по вкладам не маркетов ( не из списка)
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
Ниже третий скрипт- аналогичный скрипту 2 только вклады маркетов
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
    WHERE prod_name_res  IN (
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
 
 
Что требуется!
 
Мне нужен объем нового привлечения по срокам с 16.10. по 18.10
12:12
с разрезе сроков
12:12
На 3 мес. выделить объем по ставке РК,
 
Как мы это будем делать:
Будем брать снимок баланса на 18.10.2025
ЗАПОМНИМ что с 1 октября по 15 октября включительно рекламные ставки на 3 месяца были такими как в коде
DECLARE @rate_AT_THE_END TABLE (r decimal(9,6)); 
INSERT INTO @rate_AT_THE_END VALUES (0.165), (0.163);
 
DECLARE @rate_NOT_AT_THE_END TABLE (r decimal(9,6)); 
INSERT INTO @rate_NOT_AT_THE_END VALUES (0.162), (0.160);
НО С 16 октября ставки рекламные другие стали!
INSERT INTO @rate_AT_THE_END VALUES (0.170), (0.168);
INSERT INTO @rate_NOT_AT_THE_END VALUES (0.167), (0.165);
 
 
Напоминаю тебе маппинг срочности сделок
 
 
       when (termdays>=28 and termdays<=33) then 31
       when (termdays>=60 and termdays<=70) then 61
       when (termdays>=80 and termdays<=100) then 91
       when (termdays>=119 and termdays<=140) then 124
       when (termdays>=175 and termdays<=200) then 181
       when (termdays>=245 and termdays<=290) then 274
       when (termdays>=340 and termdays<=405) then 365
       when (termdays>=540 and termdays<=621) then 550
       when (termdays>=720 and termdays<=763) then 750
       when (termdays>=1090 and termdays<=1140) then 1100
       when (termdays>=1450 and termdays<=1475) then 1460
       when (termdays>=1795 and termdays<=1830) then 1825
          ELSE termdays
      END AS [Срок, дн.],
 
Нас интересует обьем открытых сделок пожалуй с 1 октября где мы как в скриптах 1-3 берем баланс снимок его и разбиваем сделки- но за каждый день ОБЬЕМЫ НОВЫХ СДЕЛОК! Без накопительного итога
Будем так выводить – ты просто в стольцаз даты в строках получаемые срочности + срочность “91 РК”
Исключать маркеты не будем но ты мне сделаешь аналогичный второй скрипт где я посмотрю структуру распределения маркетов
 
Жду от тебя кода ты понял задачу?
