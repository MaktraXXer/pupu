/* ──────────────────────────────────────────────────────
   0. Таблица целевых вкладов
─────────────────────────────────────────────────────── */
DECLARE @Products TABLE (prod_name_res NVARCHAR(255) PRIMARY KEY);
INSERT INTO @Products VALUES
('Надёжный'), ('Надёжный VIP'),
('Надёжный премиум'), ('Надёжный промо'),
('Надёжный старт');

/* набор дат */
DECLARE @RepDates TABLE (d DATE PRIMARY KEY);
INSERT INTO @RepDates VALUES
('2024-01-31'),('2024-02-29'),('2024-03-31'),('2024-04-30'),
('2024-05-31'),('2024-06-30'),('2024-07-31'),('2024-08-31'),
('2024-09-30'),('2024-10-31'),('2024-11-30'),('2024-12-31'),
('2025-01-31'),('2025-02-28'),('2025-03-31'),('2025-04-30'),
('2025-05-20');

/* ──────────────────────────────────────────────────────
   1. «Чистая» выборка + флаг target_prod  (1 = наш вклад)
─────────────────────────────────────────────────────── */
WITH base_data AS (
    SELECT
        bra.cli_id,
        bra.con_id,
        bra.dt_Rep,
        bra.SECTION_NAME,
        bra.TSEGMENTNAME,
        bra.PROD_NAME_RES,
        bra.OUT_RUB,
        CASE WHEN bra.PROD_NAME_RES IN (SELECT prod_name_res FROM @Products)
             THEN 1 ELSE 0 END AS target_prod      -- ←★
    FROM  ALM.ALM.Balance_Rest_All bra WITH (NOLOCK)
    JOIN  @RepDates rd            ON rd.d = bra.dt_Rep
    WHERE bra.section_name  IN ('Срочные','До востребования','Накопительный счёт')
      AND bra.cur           = '810'
      AND bra.od_flag       = 1
      AND bra.is_floatrate  = 0
      AND bra.ACC_ROLE      = 'LIAB'
      AND bra.AP            = 'Пассив'
      AND bra.TSEGMENTNAME IN ('ДЧБО','Розничный бизнес')
      AND bra.BLOCK_NAME    = 'Привлечение ФЛ'
      AND bra.OUT_RUB IS NOT NULL
)

/* ──────────────────────────────────────────────────────
   2. Первая дата «нашего» вклада
─────────────────────────────────────────────────────── */
, first_touch AS (
    SELECT cli_id,
           MIN(dt_Rep) AS first_dt_rep
    FROM   base_data
    WHERE  target_prod = 1
    GROUP BY cli_id
)

/* ──────────────────────────────────────────────────────
   3. Месячная метка generation
─────────────────────────────────────────────────────── */
, generation AS (
    SELECT cli_id,
           first_dt_rep,
           FORMAT(first_dt_rep,'yyyy-MM') AS generation
    FROM first_touch
)

/* ──────────────────────────────────────────────────────
   4. Признак pure_only  (нет ни одного «чужого» вклада)
─────────────────────────────────────────────────────── */
, product_mix AS (
    SELECT  cli_id,
            CASE WHEN SUM(CASE WHEN target_prod = 0 THEN 1 END) = 0
                 THEN 1 ELSE 0 END AS pure_only
    FROM base_data
    GROUP BY cli_id
)

/* ──────────────────────────────────────────────────────
   5. Финальная витрина
─────────────────────────────────────────────────────── */
, cte AS (
    SELECT  bd.dt_Rep,
            gen.generation,
            pm.pure_only,
            bd.SECTION_NAME,
            bd.TSEGMENTNAME,
            bd.PROD_NAME_RES,
            bd.con_id,
            bd.OUT_RUB
    FROM       base_data  bd
    JOIN       generation gen ON gen.cli_id = bd.cli_id
    JOIN       product_mix pm  ON pm.cli_id  = bd.cli_id
    /* если нужно – разкомментируйте, чтобы отрезать движения ДО открытия */
    -- AND bd.dt_Rep >= gen.first_dt_rep
)

/* ──────────────────────────────────────────────────────
   6. Агрегация
─────────────────────────────────────────────────────── */
SELECT
    dt_Rep,
    generation,
    pure_only,                    -- 1 = только продукты из списка
    SECTION_NAME,
    TSEGMENTNAME,
    PROD_NAME_RES,
    SUM(OUT_RUB)           AS sum_OUT_RUB,
    COUNT(DISTINCT con_id) AS count_CON_ID
FROM cte
GROUP BY
    dt_Rep, generation, pure_only,
    SECTION_NAME, TSEGMENTNAME, PROD_NAME_RES
ORDER BY
    dt_Rep, SECTION_NAME, TSEGMENTNAME, PROD_NAME_RES;
