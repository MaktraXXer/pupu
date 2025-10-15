Согласен — во 2-м и 3-м скриптах надо жёстко исключать все con_id, которые попали в скрипт 1 (РК-вклады), а уже из остатка делить по продуктам. Ниже — исправленные версии 2 и 3. Они:
	•	сначала строят набор rk_ids по тем же правилам, что и в скрипте 1 (ставки 16.5/16.3 для AT_THE_END и 16.2/16.0 для остальных; term_days 80–100);
	•	затем берут базу без этих con_id (ANTI JOIN через NOT EXISTS);
	•	дальше делают ваш нужный срез по prod_name_res (NOT IN — для скрипта 2; IN — для скрипта 3);
	•	считают кумулятив: cnt_deposits_cum, cnt_cli_cum, sum_out_rub_cum, sum_out_rub_cum_bln.

⸻

✅ Скрипт 2 (НЕ-РК, исключить список продуктов)

DECLARE @dt_rep       date         = '2025-10-10';
DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

-- РК-ставки для скрипта 1 (исключаемых договоров)
DECLARE @rate_AT_THE_END TABLE (r decimal(9,6));  INSERT INTO @rate_AT_THE_END VALUES (0.165),(0.163);
DECLARE @rate_NOT_AT_THE_END TABLE (r decimal(9,6));  INSERT INTO @rate_NOT_AT_THE_END VALUES (0.162),(0.160);

DECLARE @month_start date = DATEFROMPARTS(YEAR(@dt_rep), MONTH(@dt_rep), 1);

WITH base AS (  -- общая база
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
    WHERE t.dt_rep = @dt_rep
      AND t.section_name = @section_name
      AND t.block_name   = @block_name
      AND (@od_only = 0 OR t.od_flag = 1)
      AND t.cur          = @cur
      AND t.acc_role     = @acc_role
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0
      AND t.dt_open BETWEEN @month_start AND @dt_rep
      AND t.term_days BETWEEN 80 AND 100
),
-- == набор договоров РК (как в скрипте 1) ==
rk_filt AS (
    SELECT * FROM base b
    WHERE b.conv = 'AT_THE_END'
      AND b.rate_con IN (SELECT r FROM @rate_AT_THE_END)
    UNION ALL
    SELECT * FROM base b
    WHERE (b.conv <> 'AT_THE_END')  -- NULL сюда не попадёт => не будет считаться РК
      AND b.rate_con IN (SELECT r FROM @rate_NOT_AT_THE_END)
),
rk_by_con AS (  -- схлопываем РК по договору
    SELECT dt_open_d, con_id
    FROM rk_filt
    GROUP BY dt_open_d, con_id
),
rk_ids AS (     -- distinct con_id РК
    SELECT DISTINCT con_id FROM rk_by_con
),
-- == остаток НЕ-РК: выкидываем РК-договоры ==
nonrk_base AS (
    SELECT b.*
    FROM base b
    WHERE NOT EXISTS (SELECT 1 FROM rk_ids r WHERE r.con_id = b.con_id)
),
-- == фильтр скрипта 2: исключить указанные продукты ==
nonrk_filtered AS (
    SELECT *
    FROM nonrk_base
    WHERE prod_name_res NOT IN (
        N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
        N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2', N'Надёжный T2',
        N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
    )
),
by_con AS (  -- схлопываем остаток по договору
    SELECT dt_open_d, con_id, MIN(cli_id) AS cli_id, SUM(out_rub) AS out_rub
    FROM nonrk_filtered
    GROUP BY dt_open_d, con_id
),
cal AS (      -- календарь
    SELECT TOP (DATEDIFF(DAY, @month_start, @dt_rep) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @month_start) AS open_date
    FROM master..spt_values
)
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)                            AS cnt_deposits_cum,
    COUNT(DISTINCT b.cli_id)                            AS cnt_cli_cum,
    SUM(b.out_rub)                                      AS sum_out_rub_cum,
    CAST(SUM(b.out_rub) / 1e9 AS decimal(18,6))         AS sum_out_rub_cum_bln
FROM cal c
LEFT JOIN by_con b
       ON b.dt_open_d <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date;


⸻

✅ Скрипт 3 (НЕ-РК, только указанные продукты)

DECLARE @dt_rep       date         = '2025-10-10';
DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

-- РК-ставки (для исключаемого множества)
DECLARE @rate_AT_THE_END TABLE (r decimal(9,6));  INSERT INTO @rate_AT_THE_END VALUES (0.165),(0.163);
DECLARE @rate_NOT_AT_THE_END TABLE (r decimal(9,6));  INSERT INTO @rate_NOT_AT_THE_END VALUES (0.162),(0.160);

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
    WHERE t.dt_rep = @dt_rep
      AND t.section_name = @section_name
      AND t.block_name   = @block_name
      AND (@od_only = 0 OR t.od_flag = 1)
      AND t.cur          = @cur
      AND t.acc_role     = @acc_role
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0
      AND t.dt_open BETWEEN @month_start AND @dt_rep
      AND t.term_days BETWEEN 80 AND 100
),
-- == множество РК по правилам скрипта 1 ==
rk_filt AS (
    SELECT * FROM base b
    WHERE b.conv = 'AT_THE_END'
      AND b.rate_con IN (SELECT r FROM @rate_AT_THE_END)
    UNION ALL
    SELECT * FROM base b
    WHERE (b.conv <> 'AT_THE_END')
      AND b.rate_con IN (SELECT r FROM @rate_NOT_AT_THE_END)
),
rk_by_con AS (
    SELECT dt_open_d, con_id
    FROM rk_filt
    GROUP BY dt_open_d, con_id
),
rk_ids AS (
    SELECT DISTINCT con_id FROM rk_by_con
),
-- == остаток НЕ-РК ==
nonrk_base AS (
    SELECT b.*
    FROM base b
    WHERE NOT EXISTS (SELECT 1 FROM rk_ids r WHERE r.con_id = b.con_id)
),
-- == фильтр скрипта 3: оставить только указанные продукты ==
nonrk_filtered AS (
    SELECT *
    FROM nonrk_base
    WHERE prod_name_res IN (
        N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
        N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2', N'Надёжный T2',
        N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
    )
),
by_con AS (
    SELECT dt_open_d, con_id, MIN(cli_id) AS cli_id, SUM(out_rub) AS out_rub
    FROM nonrk_filtered
    GROUP BY dt_open_d, con_id
),
cal AS (
    SELECT TOP (DATEDIFF(DAY, @month_start, @dt_rep) + 1)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @month_start) AS open_date
    FROM master..spt_values
)
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)                            AS cnt_deposits_cum,
    COUNT(DISTINCT b.cli_id)                            AS cnt_cli_cum,
    SUM(b.out_rub)                                      AS sum_out_rub_cum,
    CAST(SUM(b.out_rub) / 1e9 AS decimal(18,6))         AS sum_out_rub_cum_bln
FROM cal c
LEFT JOIN by_con b
       ON b.dt_open_d <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date;


⸻

Если хочешь, объединим их в один запрос, который отдаёт три набора с колонкой slice ('RK' | 'nonRK_excluded_products' | 'nonRK_only_products') — будет удобнее сравнивать на одном графике/сводной.
