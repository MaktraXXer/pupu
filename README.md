/* ───────── 0. Целевые вклады ───────── */
DECLARE @Products TABLE(prod_name_res NVARCHAR(255) PRIMARY KEY);
INSERT INTO @Products VALUES
('Надёжный'), ('Надёжный VIP'),
('Надёжный премиум'), ('Надёжный промо'),
('Надёжный старт');

/* ───────── 1. Читаем подходящие строки во временную таблицу ───────── */
SELECT
    cli_id,
    con_id,
    dt_Rep,
    SECTION_NAME,
    TSEGMENTNAME,
    PROD_NAME_RES,
    OUT_RUB,
    CASE WHEN PROD_NAME_RES IN (SELECT prod_name_res FROM @Products)
         THEN 1 ELSE 0 END AS target_prod
INTO   #bd
FROM   ALM.ALM.Balance_Rest_All WITH (NOLOCK)
WHERE  dt_Rep BETWEEN '2024-01-01' AND '2025-05-31'
  AND  dt_Rep IN ('2024-01-31','2024-02-29','2024-03-31','2024-04-30',
                  '2024-05-31','2024-06-30','2024-07-31','2024-08-31',
                  '2024-09-30','2024-10-31','2024-11-30','2024-12-31',
                  '2025-01-31','2025-02-28','2025-03-31','2025-04-30',
                  '2025-05-20')
  AND  section_name  IN ('Срочные','До востребования','Накопительный счёт')
  AND  cur           = '810'
  AND  od_flag       = 1
  AND  is_floatrate  = 0
  AND  ACC_ROLE      = 'LIAB'
  AND  AP            = 'Пассив'
  AND  TSEGMENTNAME IN ('ДЧБО','Розничный бизнес')
  AND  BLOCK_NAME    = 'Привлечение ФЛ'
  AND  OUT_RUB IS NOT NULL;

CREATE CLUSTERED INDEX ix_bd_cli_dt ON #bd (cli_id, dt_Rep);

/* ───────── 2. Оконные функции — всё в один проход ───────── */
WITH step1 AS (
    SELECT *,
        /* дата первого целевого вклада */
        MIN(CASE WHEN target_prod = 1 THEN dt_Rep END)
            OVER (PARTITION BY cli_id)                    AS first_target_dt
    FROM #bd
)
, step2 AS (
    SELECT *,
        /* есть ли dépôt ДО generation? 1 = был */
        CASE WHEN MAX(CASE WHEN dt_Rep < first_target_dt THEN 1 END)
                 OVER (PARTITION BY cli_id) = 1
             THEN 1 ELSE 0 END                            AS had_deposit_before,

        /* квартальная метка */
        CONCAT(DATEPART(year , first_target_dt),
               'Q',
               DATEPART(quarter, first_target_dt))        AS vintage_qtr
    FROM step1
    WHERE first_target_dt IS NOT NULL    -- исключаем клиентов без целевого вклада
)

/* ───────── 3. Агрегация ───────── */
SELECT
    dt_Rep,
    CONVERT(char(7), first_target_dt, 120) AS generation,   -- YYYY-MM
    vintage_qtr,                                            -- YYYYQn
    had_deposit_before,                                     -- 0 = «совсем новые»
    SECTION_NAME,
    TSEGMENTNAME,
    PROD_NAME_RES,
    SUM(OUT_RUB)           AS sum_OUT_RUB,
    COUNT(DISTINCT con_id) AS count_CON_ID
FROM step2
GROUP BY
    dt_Rep,
    CONVERT(char(7), first_target_dt, 120),
    vintage_qtr,
    had_deposit_before,
    SECTION_NAME,
    TSEGMENTNAME,
    PROD_NAME_RES
ORDER BY
    dt_Rep, SECTION_NAME, TSEGMENTNAME, PROD_NAME_RES;

DROP TABLE #bd;
