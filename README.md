Да, понял задачу. Ниже даю готовые T-SQL скрипты под ваш кейс — один «основной» (без исключения маркетов), и второй — фильтр только «маркетов», чтобы посмотреть их структуру. Оба скрипта:
	•	Берут один снимок баланса на 2025-10-18 (@dt_rep).
	•	Считают дневные объёмы новых сделок (без накопительного итога) по dt_open с 2025-10-01 по 2025-10-18.
	•	Делают маппинг срочности по вашему правилу.
	•	В строках дают «срочности» + отдельную строку 91 РК (только трёхмесячные по рекламным ставкам), в столбцах — даты (динамический PIVOT).
	•	Логика РК-ставок для 91 дня меняется с 16.10:
— 01–15.10: AT_THE_END ∈ {0.165, 0.163}, NOT_AT_THE_END ∈ {0.162, 0.160}
— 16–18.10: AT_THE_END ∈ {0.170, 0.168}, NOT_AT_THE_END ∈ {0.167, 0.165}
	•	Дедуп по договору в день открытия (агрегация по con_id).

⸻

Скрипт A — все сделки (маркет не исключаем), плюс строка «91 РК»

/* =============================== *
   ПАРАМЕТРЫ
*  =============================== */
DECLARE @dt_rep       date         = '2025-10-18';        -- один снимок баланса
DECLARE @date_from    date         = '2025-10-01';
DECLARE @date_to      date         = '2025-10-18';
DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

/* ——— РК-ставки: октябрь разбит на 2 периода ——— */
DECLARE @rate_old_AT_THE_END       TABLE (r decimal(9,6)); INSERT INTO @rate_old_AT_THE_END       VALUES (0.165),(0.163);  -- 01–15.10
DECLARE @rate_old_NOT_AT_THE_END   TABLE (r decimal(9,6)); INSERT INTO @rate_old_NOT_AT_THE_END   VALUES (0.162),(0.160);
DECLARE @rate_new_AT_THE_END       TABLE (r decimal(9,6)); INSERT INTO @rate_new_AT_THE_END       VALUES (0.170),(0.168);  -- 16–18.10
DECLARE @rate_new_NOT_AT_THE_END   TABLE (r decimal(9,6)); INSERT INTO @rate_new_NOT_AT_THE_END   VALUES (0.167),(0.165);

DECLARE @rk_switch_from date = '2025-10-16';   -- дата смены рекламных ставок (включительно)

IF OBJECT_ID('tempdb..#tall') IS NOT NULL DROP TABLE #tall;

/* =============================== *
   ОСНОВНАЯ БАЗА: один снимок, окно открытий
*  =============================== */
WITH base AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.conv,
        t.termdays
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE
        t.dt_rep       = @dt_rep
        AND t.section_name = @section_name
        AND t.block_name   = @block_name
        AND (@od_only = 0 OR t.od_flag = 1)
        AND t.cur          = @cur
        AND t.acc_role     = @acc_role
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
        AND t.dt_open BETWEEN @date_from AND @date_to
),
/* Дедуп по договору на дату открытия */
by_con AS (
    SELECT
        dt_open_d,
        con_id,
        MIN(cli_id)  AS cli_id,
        SUM(out_rub) AS out_rub,
        MIN(rate_con) AS rate_con,             -- ставка на договор (если несколько строк)
        MIN(conv)     AS conv,
        MIN(termdays) AS termdays
    FROM base
    GROUP BY dt_open_d, con_id
),
/* Маппинг срочности */
mapped AS (
    SELECT
        dt_open_d,
        con_id,
        cli_id,
        out_rub,
        rate_con,
        conv,
        CASE
            WHEN (termdays>=28  AND termdays<=33)   THEN 31
            WHEN (termdays>=60  AND termdays<=70)   THEN 61
            WHEN (termdays>=80  AND termdays<=100)  THEN 91
            WHEN (termdays>=119 AND termdays<=140)  THEN 124
            WHEN (termdays>=175 AND termdays<=200)  THEN 181
            WHEN (termdays>=245 AND termdays<=290)  THEN 274
            WHEN (termdays>=340 AND termdays<=405)  THEN 365
            WHEN (termdays>=540 AND termdays<=621)  THEN 550
            WHEN (termdays>=720 AND termdays<=763)  THEN 750
            WHEN (termdays>=1090 AND termdays<=1140) THEN 1100
            WHEN (termdays>=1450 AND termdays<=1475) THEN 1460
            WHEN (termdays>=1795 AND termdays<=1830) THEN 1825
            ELSE termdays
        END AS term_bucket
    FROM by_con
),
/* Флаг «91 РК»: только 91-дневные и только если ставка попадает в набор РК, причём набор зависит от даты открытия */
flag_91rk AS (
    SELECT
        m.*,
        CASE
            WHEN m.term_bucket = 91 AND m.conv = 'AT_THE_END' AND m.dt_open_d <  @rk_switch_from AND m.rate_con IN (SELECT r FROM @rate_old_AT_THE_END)     THEN 1
            WHEN m.term_bucket = 91 AND (m.conv <> 'AT_THE_END')            AND m.dt_open_d <  @rk_switch_from AND m.rate_con IN (SELECT r FROM @rate_old_NOT_AT_THE_END) THEN 1

            WHEN m.term_bucket = 91 AND m.conv = 'AT_THE_END' AND m.dt_open_d >= @rk_switch_from AND m.rate_con IN (SELECT r FROM @rate_new_AT_THE_END)     THEN 1
            WHEN m.term_bucket = 91 AND (m.conv <> 'AT_THE_END')            AND m.dt_open_d >= @rk_switch_from AND m.rate_con IN (SELECT r FROM @rate_new_NOT_AT_THE_END) THEN 1
            ELSE 0
        END AS is_91_rk
    FROM mapped m
),
/* Высота (tall): одна строка = [СтрокаОтчёта, Дата, Объём]
   — строкаОтчёта = <числовая срочность> ИЛИ специальная строка '91 РК' */
tall AS (
    /* обычные срочности (в т.ч. 91 общий) */
    SELECT
        CAST(flag_91rk.term_bucket AS nvarchar(20)) AS row_label,
        flag_91rk.dt_open_d AS col_date,
        SUM(flag_91rk.out_rub) AS out_rub
    FROM flag_91rk
    GROUP BY flag_91rk.term_bucket, flag_91rk.dt_open_d

    UNION ALL

    /* специальная строка только по 91 РК */
    SELECT
        N'91 РК' AS row_label,
        f.dt_open_d AS col_date,
        SUM(f.out_rub) AS out_rub
    FROM flag_91rk f
    WHERE f.term_bucket = 91 AND f.is_91_rk = 1
    GROUP BY f.dt_open_d
)
SELECT *
INTO #tall
FROM tall;

/* =============================== *
   ДИНАМИЧЕСКИЙ PIVOT: даты — по столбцам
*  =============================== */
DECLARE @cols nvarchar(max) =
    STUFF((
        SELECT DISTINCT ', ' + QUOTENAME(CONVERT(varchar(10), col_date, 120))
        FROM #tall
        ORDER BY ', ' + QUOTENAME(CONVERT(varchar(10), col_date, 120))
        FOR XML PATH(''), TYPE
    ).value('.', 'nvarchar(max)'), 1, 2, '');

DECLARE @sql nvarchar(max) = N'
SELECT row_label, ' + @cols + N'
FROM (
    SELECT row_label,
           CONVERT(varchar(10), col_date, 120) AS col_date,
           out_rub
    FROM #tall
) s
PIVOT (
    SUM(out_rub) FOR col_date IN (' + @cols + N')
) p
ORDER BY
    CASE WHEN row_label = N''91 РК'' THEN 999999 ELSE TRY_CONVERT(int, row_label) END, row_label;';

EXEC sp_executesql @sql;

Что получите: таблицу, где строки — «31, 61, 91, 124, …, 1825, 91 РК», а столбцы — даты 2025-10-01..18; в ячейках — объёмы новых открытий (по out_rub) на соответствующую дату. Строка 91 РК содержит только «трёхмесячные» сделки, удовлетворяющие рекламным ставкам с учётом смены ставок 16.10.

⸻

Скрипт B — только «маркет»-продукты (для анализа структуры маркетов)

/* Те же параметры, тот же снимок и окно дат */
DECLARE @dt_rep       date         = '2025-10-18';
DECLARE @date_from    date         = '2025-10-01';
DECLARE @date_to      date         = '2025-10-18';
DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

DECLARE @rate_old_AT_THE_END       TABLE (r decimal(9,6)); INSERT INTO @rate_old_AT_THE_END       VALUES (0.165),(0.163);
DECLARE @rate_old_NOT_AT_THE_END   TABLE (r decimal(9,6)); INSERT INTO @rate_old_NOT_AT_THE_END   VALUES (0.162),(0.160);
DECLARE @rate_new_AT_THE_END       TABLE (r decimal(9,6)); INSERT INTO @rate_new_AT_THE_END       VALUES (0.170),(0.168);
DECLARE @rate_new_NOT_AT_THE_END   TABLE (r decimal(9,6)); INSERT INTO @rate_new_NOT_AT_THE_END   VALUES (0.167),(0.165);

DECLARE @rk_switch_from date = '2025-10-16';

IF OBJECT_ID('tempdb..#tall_mrk') IS NOT NULL DROP TABLE #tall_mrk;

/* Набор маркет-продуктов (как у вас в примерах) */
DECLARE @market TABLE (name nvarchar(100));
INSERT INTO @market(name) VALUES
 (N'Надёжный прайм'), (N'Надёжный VIP'), (N'Надёжный премиум'),
 (N'Надёжный промо'), (N'Надёжный старт'),
 (N'Надёжный Т2'), (N'Надёжный T2'),
 (N'Надёжный Мегафон'), (N'Надёжный процент'),
 (N'Надёжный'), (N'ДОМа надёжно'), (N'Всё в ДОМ');

WITH base AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.conv,
        t.termdays,
        t.prod_name_res
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE
        t.dt_rep       = @dt_rep
        AND t.section_name = @section_name
        AND t.block_name   = @block_name
        AND (@od_only = 0 OR t.od_flag = 1)
        AND t.cur          = @cur
        AND t.acc_role     = @acc_role
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
        AND t.dt_open BETWEEN @date_from AND @date_to
        AND EXISTS (SELECT 1 FROM @market m WHERE m.name = t.prod_name_res)  -- только маркеты
),
by_con AS (
    SELECT
        dt_open_d,
        con_id,
        MIN(cli_id)  AS cli_id,
        SUM(out_rub) AS out_rub,
        MIN(rate_con) AS rate_con,
        MIN(conv)     AS conv,
        MIN(termdays) AS termdays
    FROM base
    GROUP BY dt_open_d, con_id
),
mapped AS (
    SELECT
        dt_open_d,
        con_id,
        cli_id,
        out_rub,
        rate_con,
        conv,
        CASE
            WHEN (termdays>=28  AND termdays<=33)   THEN 31
            WHEN (termdays>=60  AND termdays<=70)   THEN 61
            WHEN (termdays>=80  AND termdays<=100)  THEN 91
            WHEN (termdays>=119 AND termdays<=140)  THEN 124
            WHEN (termdays>=175 AND termdays<=200)  THEN 181
            WHEN (termdays>=245 AND termdays<=290)  THEN 274
            WHEN (termdays>=340 AND termdays<=405)  THEN 365
            WHEN (termdays>=540 AND termdays<=621)  THEN 550
            WHEN (termdays>=720 AND termdays<=763)  THEN 750
            WHEN (termdays>=1090 AND termdays<=1140) THEN 1100
            WHEN (termdays>=1450 AND termdays<=1475) THEN 1460
            WHEN (termdays>=1795 AND termdays<=1830) THEN 1825
            ELSE termdays
        END AS term_bucket
    FROM by_con
),
flag_91rk AS (
    SELECT
        m.*,
        CASE
            WHEN m.term_bucket = 91 AND m.conv = 'AT_THE_END' AND m.dt_open_d <  @rk_switch_from AND m.rate_con IN (SELECT r FROM @rate_old_AT_THE_END)     THEN 1
            WHEN m.term_bucket = 91 AND (m.conv <> 'AT_THE_END')            AND m.dt_open_d <  @rk_switch_from AND m.rate_con IN (SELECT r FROM @rate_old_NOT_AT_THE_END) THEN 1

            WHEN m.term_bucket = 91 AND m.conv = 'AT_THE_END' AND m.dt_open_d >= @rk_switch_from AND m.rate_con IN (SELECT r FROM @rate_new_AT_THE_END)     THEN 1
            WHEN m.term_bucket = 91 AND (m.conv <> 'AT_THE_END')            AND m.dt_open_d >= @rk_switch_from AND m.rate_con IN (SELECT r FROM @rate_new_NOT_AT_THE_END) THEN 1
            ELSE 0
        END AS is_91_rk
    FROM mapped m
),
tall AS (
    SELECT CAST(term_bucket AS nvarchar(20)) AS row_label,
           dt_open_d AS col_date,
           SUM(out_rub) AS out_rub
    FROM flag_91rk
    GROUP BY term_bucket, dt_open_d

    UNION ALL

    SELECT N'91 РК' AS row_label,
           f.dt_open_d AS col_date,
           SUM(f.out_rub) AS out_rub
    FROM flag_91rk f
    WHERE f.term_bucket = 91 AND f.is_91_rk = 1
    GROUP BY f.dt_open_d
)
SELECT *
INTO #tall_mrk
FROM tall;

/* Динамический PIVOT по датам */
DECLARE @cols_m nvarchar(max) =
    STUFF((
        SELECT DISTINCT ', ' + QUOTENAME(CONVERT(varchar(10), col_date, 120))
        FROM #tall_mrk
        ORDER BY ', ' + QUOTENAME(CONVERT(varchar(10), col_date, 120))
        FOR XML PATH(''), TYPE
    ).value('.', 'nvarchar(max)'), 1, 2, '');

DECLARE @sql_m nvarchar(max) = N'
SELECT row_label, ' + @cols_m + N'
FROM (
    SELECT row_label,
           CONVERT(varchar(10), col_date, 120) AS col_date,
           out_rub
    FROM #tall_mrk
) s
PIVOT (
    SUM(out_rub) FOR col_date IN (' + @cols_m + N')
) p
ORDER BY
    CASE WHEN row_label = N''91 РК'' THEN 999999 ELSE TRY_CONVERT(int, row_label) END, row_label;';

EXEC sp_executesql @sql_m;


⸻

Примечания и нюансы
	•	NULL conv в РК не учитываем (как в ваших прежних правилах): в проверках стоит m.conv = 'AT_THE_END' и (m.conv <> 'AT_THE_END'), так что NULL в РК не попадёт и уйдёт в «обычные 91».
	•	Если хотите видеть ноль для дат/строк без сделок: текущий PIVOT вернёт NULL, можно оборачивать в ISNULL на чтении результата, либо пост-процессингом в отчёте.
	•	Если нужно отдельное «Итого» по строкам/датам — скажите, добавлю вторым UNION ALL.
	•	Список «маркетов» взял из ваших примеров. Если он расширяется — просто добавьте значения в @market.

Если хочешь, сделаю ещё третий маленький запрос-проверку: в разрезе дат 01–18.10 вывести сколько 91-дневных попало в РК-наборы старые/новые отдельно — для быстрой валидации смены ставок с 16.10.
