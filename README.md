/* 0. Список целевых вкладов */
DECLARE @Products TABLE (prod_name_res NVARCHAR(255) PRIMARY KEY);
INSERT INTO @Products VALUES
('Надёжный'), ('Надёжный VIP'), ('Надёжный премиум'),
('Надёжный промо'), ('Надёжный старт');

/* 1. Клиенты, у которых хотя бы раз был нужный вклад,
      + дата первого появления                        */
WITH first_touch AS (
    SELECT
        cli_id,
        MIN(dt_Rep) AS first_dt_rep
    FROM   ALM.ALM.Balance_Rest_All WITH (NOLOCK)
    WHERE  prod_name_res IN (SELECT prod_name_res FROM @Products)
      AND  dt_Rep IN ( '2024-01-31','2024-02-29','2024-03-31','2024-04-30',
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
      AND  OUT_RUB IS NOT NULL
    GROUP BY cli_id
)

/* 2. Добавляем колонку «винтаж-квартал» */
, vintage AS (
    SELECT
        cli_id,
        CONCAT(DATEPART(year , first_dt_rep), 'Q', DATEPART(quarter, first_dt_rep)) AS vintage_qtr
    FROM first_touch
)

/* 3. Берём **все** записи рублёвых пассивов по найденным клиентам */
, cte AS (
    SELECT
        bra.dt_Rep,
        bra.SECTION_NAME,
        bra.TSEGMENTNAME,
        bra.PROD_NAME_RES,
        bra.con_id,
        bra.OUT_RUB,
        v.vintage_qtr
    FROM ALM.ALM.Balance_Rest_All bra WITH (NOLOCK)
    JOIN vintage v
         ON v.cli_id = bra.cli_id               -- только клиенты из п.1
    WHERE bra.dt_Rep IN ( '2024-01-31','2024-02-29','2024-03-31','2024-04-30',
                          '2024-05-31','2024-06-30','2024-07-31','2024-08-31',
                          '2024-09-30','2024-10-31','2024-11-30','2024-12-31',
                          '2025-01-31','2025-02-28','2025-03-31','2025-04-30',
                          '2025-05-20')
      AND bra.section_name  IN ('Срочные','До востребования','Накопительный счёт')
      AND bra.cur           = '810'
      AND bra.od_flag       = 1
      AND bra.is_floatrate  = 0
      AND bra.ACC_ROLE      = 'LIAB'
      AND bra.AP            = 'Пассив'
      AND bra.TSEGMENTNAME IN ('ДЧБО','Розничный бизнес')
      AND bra.BLOCK_NAME    = 'Привлечение ФЛ'
      AND bra.OUT_RUB IS NOT NULL
)

/* 4. Итоговая агрегация с винтажом */
SELECT
    dt_Rep,
    vintage_qtr,
    SECTION_NAME,
    TSEGMENTNAME,
    PROD_NAME_RES,
    SUM(OUT_RUB)           AS sum_OUT_RUB,
    COUNT(DISTINCT con_id) AS count_CON_ID
FROM cte
GROUP BY
    dt_Rep, vintage_qtr, SECTION_NAME, TSEGMENTNAME, PROD_NAME_RES
ORDER BY
    dt_Rep, SECTION_NAME, TSEGMENTNAME, PROD_NAME_RES;
