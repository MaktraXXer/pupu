/* ──────────────────────────────────────────────────────
   Создать таблицу для сохранения результатов винтаж-отчёта
   Схема: alm_test.dbo
────────────────────────────────────────────────────── */
IF OBJECT_ID('alm_test.dbo.fu_vintage_results', 'U') IS NOT NULL
    DROP TABLE alm_test.dbo.fu_vintage_results;
GO

CREATE TABLE alm_test.dbo.fu_vintage_results
(
    -- ключ/дата отчётного среза
    dt_rep             DATE            NOT NULL,

    -- клиент
    cli_id             BIGINT          NOT NULL,        -- если у вас GUID или NVARCHAR → замените тип
    generation         CHAR(7)         NOT NULL,        -- 'YYYY-MM'
    vintage_qtr        CHAR(6)         NOT NULL,        -- 'YYYYQn'
    had_deposit_before BIT             NOT NULL,        -- 0/1

    -- бизнес-атрибуты
    section_name       NVARCHAR(50)    NOT NULL,
    tsegmentname       NVARCHAR(50)    NOT NULL,
    prod_name_res      NVARCHAR(100)   NOT NULL,

    -- метрики
    sum_out_rub        DECIMAL(20,2)   NOT NULL,
    count_con_id       INT             NOT NULL,
    rate_obiem         DECIMAL(20,2)   NOT NULL,

    -- служебное
    load_timestamp     DATETIME2       NOT NULL
        CONSTRAINT DF_fu_vintage_results_loadts DEFAULT (SYSUTCDATETIME()),

    CONSTRAINT PK_fu_vintage_results
        PRIMARY KEY CLUSTERED (dt_rep, cli_id, prod_name_res)
);
GO

/* не обязательные, но полезные индексы */

-- по дате и поколению: удобно фильтровать по срезу и винтажу
CREATE INDEX IX_fu_vintage_results_rep_gen
ON alm_test.dbo.fu_vintage_results (dt_rep, generation);

-- по кварталу и наличию ранних депозитов (для быстрых сегментов)
CREATE INDEX IX_fu_vintage_results_vint_had
ON alm_test.dbo.fu_vintage_results (vintage_qtr, had_deposit_before);
GO

Я запущу этот код

дальше я запущу такой код

/* ───────── 0. Целевые вклады ───────── */
DROP TABLE IF EXISTS #bd;
DECLARE @Products TABLE (prod_name_res NVARCHAR(255) PRIMARY KEY);
INSERT INTO @Products VALUES
('Надёжный'), ('Надёжный VIP'),
('Надёжный премиум'), ('Надёжный промо'),
('Надёжный старт');

/* ───────── 1. выгружаем данные во временную таблицу ───────── */
SELECT
    cli_id,
    con_id,
    dt_rep,
    section_name,
    tsegmentname,
    prod_name_res,
    out_rub,
    rate_con,
    CASE WHEN prod_name_res IN (SELECT prod_name_res FROM @Products)
         THEN 1 ELSE 0 END AS target_prod
INTO  #bd
FROM  ALM.ALM.Balance_Rest_All WITH (NOLOCK)
WHERE dt_rep BETWEEN '2024-01-01' AND '2025-05-31'
  AND dt_rep IN ('2024-01-31','2024-02-29','2024-03-31','2024-04-30',
                 '2024-05-31','2024-06-30','2024-07-31','2024-08-31',
                 '2024-09-30','2024-10-31','2024-11-30','2024-12-31',
                 '2025-01-31','2025-02-28','2025-03-31','2025-04-30',
                 '2025-05-20')
  AND section_name  IN ('Срочные','До востребования','Накопительный счёт')
  AND cur           = '810'
  AND od_flag       = 1
  AND is_floatrate  = 0
  AND acc_role      = 'LIAB'
  AND ap            = 'Пассив'
  AND tsegmentname IN ('ДЧБО','Розничный бизнес')
  AND block_name    = 'Привлечение ФЛ'
  AND out_rub IS NOT NULL;

CREATE CLUSTERED INDEX ix_bd_cli_dt ON #bd (cli_id, dt_rep);

/* ───────── 2. оконные функции ───────── */
WITH step1 AS (
    SELECT *,
           MIN(CASE WHEN target_prod = 1 THEN dt_rep END)
               OVER(PARTITION BY cli_id) AS first_target_dt
    FROM #bd
),
step2 AS (
    SELECT *,
           CASE WHEN MAX(CASE WHEN dt_rep < first_target_dt THEN 1 END)
                    OVER(PARTITION BY cli_id) = 1
                THEN 1 ELSE 0 END                         AS had_deposit_before,
           CONCAT(DATEPART(year, first_target_dt), 'Q',
                  DATEPART(quarter, first_target_dt))     AS vintage_qtr
    FROM step1
    WHERE first_target_dt IS NOT NULL
)

/* ───────── 3. агрегирование с cli_id ───────── */
SELECT
    dt_rep,
    cli_id,
    CONVERT(char(7), first_target_dt, 120) AS generation,   -- YYYY-MM
    vintage_qtr,
    had_deposit_before,
    section_name,
    tsegmentname,
    prod_name_res,
    SUM(out_rub)                  AS sum_out_rub,
    COUNT(DISTINCT con_id)        AS count_con_id,
    SUM(out_rub * rate_con)       AS rate_obiem
FROM step2
GROUP BY
    dt_rep,
    cli_id,                       -- ← добавили
    CONVERT(char(7), first_target_dt, 120),
    vintage_qtr,
    had_deposit_before,
    section_name,
    tsegmentname,
    prod_name_res
ORDER BY
    dt_rep,
    cli_id,
    section_name,
    tsegmentname,
    prod_name_res;

-- DROP TABLE #bd;
