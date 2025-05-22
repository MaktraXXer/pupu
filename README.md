/* ------------------------------------------------------
   Задаём список интересующих продуктов – удобнее через
   табличную переменную, чтобы легко менять состав.
------------------------------------------------------ */
DECLARE @Products TABLE (prod_name_res NVARCHAR(255) PRIMARY KEY);
INSERT INTO @Products VALUES
('Продукт А'), ('Продукт B'), ('Продукт C');       -- допишите свои названия

/* -------------------------------------------
   1-й CTE: находим всех нужных клиентов
------------------------------------------- */
WITH clients_with_product AS (
    SELECT DISTINCT con_id
    FROM   ALM.ALM.Balance_Rest_All WITH (NOLOCK)
    WHERE  prod_name_res IN (SELECT prod_name_res FROM @Products)
           AND dt_Rep IN ( '2024-01-31','2024-02-29','2024-03-31','2024-04-30',
                           '2024-05-31','2024-06-30','2024-07-31','2024-08-31',
                           '2024-09-30','2024-10-31','2024-11-30','2024-12-31',
                           '2025-01-31','2025-02-28','2025-03-31')
           /* базовые ограничения */
           AND cur          = 810
           AND od_flag      = 1
           AND is_floatrate = 0
           AND TSEGMENTNAME IN ('ДЧБО','Розничный бизнес')
           AND BLOCK_NAME   = 'Привлечение ФЛ'
)

/* -------------------------------------------
   2-й CTE: берём все записи, которые и так
   нужны вам, но только по найденным клиентам
------------------------------------------- */
, cte AS (
    SELECT dt_Rep,
           SECTION_NAME,
           TSEGMENTNAME,
           PROD_NAME_RES,
           con_id,
           OUT_RUB
    FROM   ALM.ALM.Balance_Rest_All WITH (NOLOCK)
    WHERE  dt_Rep IN ( '2024-01-31','2024-02-29','2024-03-31','2024-04-30',
                       '2024-05-31','2024-06-30','2024-07-31','2024-08-31',
                       '2024-09-30','2024-10-31','2024-11-30','2024-12-31',
                       '2025-01-31','2025-02-28','2025-03-31')
      AND SECTION_NAME  IN ('Срочные','До востребования','Накопительный счёт')
      AND cur           = 810
      AND od_flag       = 1
      AND is_floatrate  = 0
      AND TSEGMENTNAME  IN ('ДЧБО','Розничный бизнес')
      AND BLOCK_NAME    = 'Привлечение ФЛ'
      AND PROD_NAME_RES IS NOT NULL
      AND OUT_RUB       IS NOT NULL
      AND con_id IN (SELECT con_id FROM clients_with_product)   -- главное условие
)

/* -------------------------------------------
   Итоговая агрегация
------------------------------------------- */
SELECT  dt_Rep,
        SECTION_NAME,
        TSEGMENTNAME,
        PROD_NAME_RES,
        SUM(OUT_RUB)          AS sum_OUT_RUB,
        COUNT(DISTINCT con_id) AS count_CON_ID
FROM    cte
GROUP BY dt_Rep, SECTION_NAME, TSEGMENTNAME, PROD_NAME_RES
ORDER BY dt_Rep, SECTION_NAME, TSEGMENTNAME, PROD_NAME_RES;
