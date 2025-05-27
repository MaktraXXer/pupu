Ниже — полный, самодостаточный T-SQL-скрипт «от нуля до загрузки» :

* создаёт таблицу-приёмник **alm\_test.dbo.fu\_vintage\_results** (если её ещё нет);
* выгружает данные из **ALM.ALM.Balance\_Rest\_All** во временную таблицу `#bd`;
* нормализует названия **только** для вкладов из списка `@Products`
  — берёт *самое позднее* имя по каждому `con_id`;
* рассчитывает `generation`, `vintage_qtr`, `had_deposit_before`;
* агрегирует метрики (`sum_out_rub`, `count_con_id`, `rate_obiem`);
* очищает целевой диапазон дат в приёмнике и вставляет свежие данные.

```sql
/* =======================================================================
   0. Параметры расчёта: задайте период, который пересчитываете
======================================================================= */
DECLARE @DateFrom date = '2024-01-31';
DECLARE @DateTo   date = '2025-05-20';

/* =======================================================================
   1. Таблица-приёмник (создаётся один раз)
======================================================================= */
IF OBJECT_ID('alm_test.dbo.fu_vintage_results', 'U') IS NULL
BEGIN
    CREATE TABLE alm_test.dbo.fu_vintage_results
    (
        dt_rep             date          NOT NULL,
        cli_id             bigint        NOT NULL,
        generation         char(7)       NOT NULL,   -- 'YYYY-MM'
        vintage_qtr        char(6)       NOT NULL,   -- 'YYYYQn'
        had_deposit_before bit           NOT NULL,   -- 0/1
        section_name       nvarchar(50)  NOT NULL,
        tsegmentname       nvarchar(50)  NOT NULL,
        prod_name_res      nvarchar(100) NOT NULL,
        sum_out_rub        decimal(20,2) NOT NULL,
        count_con_id       int           NOT NULL,
        rate_obiem         decimal(20,2) NOT NULL,
        load_timestamp     datetime2     NOT NULL
            CONSTRAINT DF_fu_vintage_results_loadts DEFAULT (sysutcdatetime()),
        CONSTRAINT PK_fu_vintage_results
            PRIMARY KEY CLUSTERED (dt_rep, cli_id, prod_name_res)
    );

    CREATE INDEX IX_fu_vintage_results_rep_gen
        ON alm_test.dbo.fu_vintage_results (dt_rep, generation);

    CREATE INDEX IX_fu_vintage_results_vint_had
        ON alm_test.dbo.fu_vintage_results (vintage_qtr, had_deposit_before);
END
GO

/* =======================================================================
   2. Справочник целевых вкладов «Финуслуги»
======================================================================= */
DROP TABLE IF EXISTS #bd;
DECLARE @Products TABLE (prod_name_res nvarchar(255) PRIMARY KEY);
INSERT INTO @Products VALUES
('Надёжный'), ('Надёжный VIP'),
('Надёжный премиум'), ('Надёжный промо'),
('Надёжный старт');

/* =======================================================================
   3. Выгрузка во временную таблицу
======================================================================= */
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
WHERE dt_rep BETWEEN @DateFrom AND @DateTo
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

CREATE CLUSTERED INDEX IX_bd_cli_dt ON #bd (con_id, dt_rep);  -- нужен для оконок

/* =======================================================================
   4. Нормализация названий: берём самое позднее имя по con_id
======================================================================= */
;WITH last_name AS (
    SELECT DISTINCT
           con_id,
           FIRST_VALUE(prod_name_res)
             OVER (PARTITION BY con_id ORDER BY dt_rep DESC) AS prod_latest
    FROM #bd
    WHERE target_prod = 1
)
UPDATE b
SET    b.prod_name_res = ln.prod_latest
FROM   #bd b
JOIN   last_name ln ON ln.con_id = b.con_id
WHERE  b.target_prod = 1;   -- только целевые строки правим

/* =======================================================================
   5. Расчёт generation / vintage / флагов
======================================================================= */
WITH step1 AS (
    SELECT *,
           MIN(CASE WHEN target_prod = 1 THEN dt_rep END)
               OVER (PARTITION BY cli_id) AS first_target_dt
    FROM #bd
),
step2 AS (
    SELECT *,
           CASE WHEN MAX(CASE WHEN dt_rep < first_target_dt THEN 1 END)
                    OVER (PARTITION BY cli_id) = 1
                THEN 1 ELSE 0 END                         AS had_deposit_before,
           CONCAT(DATEPART(year, first_target_dt),'Q',
                  DATEPART(quarter, first_target_dt))     AS vintage_qtr
    FROM step1
    WHERE first_target_dt IS NOT NULL      -- исключаем клиентов без вкладов ФУ
),
agg AS (
    SELECT
        dt_rep,
        cli_id,
        CONVERT(char(7), first_target_dt, 120) AS generation,
        vintage_qtr,
        had_deposit_before,
        section_name,
        tsegmentname,
        prod_name_res,
        CAST(SUM(out_rub)            AS decimal(20,2)) AS sum_out_rub,
        COUNT(DISTINCT con_id)                       AS count_con_id,
        CAST(SUM(out_rub * rate_con) AS decimal(20,2)) AS rate_obiem
    FROM step2
    GROUP BY
        dt_rep,
        cli_id,
        CONVERT(char(7), first_target_dt, 120),
        vintage_qtr,
        had_deposit_before,
        section_name,
        tsegmentname,
        prod_name_res
)

/* =======================================================================
   6. Перезаписываем диапазон и загружаем данные
======================================================================= */
DELETE FROM alm_test.dbo.fu_vintage_results
WHERE dt_rep BETWEEN @DateFrom AND @DateTo;

INSERT INTO alm_test.dbo.fu_vintage_results
        (dt_rep, cli_id, generation, vintage_qtr, had_deposit_before,
         section_name, tsegmentname, prod_name_res,
         sum_out_rub, count_con_id, rate_obiem)
SELECT
    dt_rep, cli_id, generation, vintage_qtr, had_deposit_before,
    section_name, tsegmentname, prod_name_res,
    sum_out_rub, count_con_id, rate_obiem
FROM agg;

DROP TABLE #bd;   -- аккуратно убираем временную таблицу
GO
```

**Проверка после загрузки**

```sql
SELECT TOP 10 * 
FROM   alm_test.dbo.fu_vintage_results
ORDER  BY load_timestamp DESC;
```

Скрипт можно запускать регулярно: достаточно менять `@DateFrom` / `@DateTo` — данные за этот интервал сначала удаляются, потом загружаются актуальные результаты с нормализованными названиями продуктов из «Финуслуг».
