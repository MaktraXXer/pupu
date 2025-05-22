/* ------------------------------------------------------------------
   0. Список интересующих вкладов
-------------------------------------------------------------------*/
DECLARE @Products TABLE (prod_name_res NVARCHAR(255) PRIMARY KEY);
INSERT INTO @Products VALUES
('Надёжный'),
('Надёжный VIP'),
('Надёжный премиум'),
('Надёжный промо'),
('Надёжный старт');

/* ------------------------------------------------------------------
   1. Отбираем все записи, удовлетворяющие бизнес-фильтрам
   (они же понадобятся и для поиска «первого появления»)
-------------------------------------------------------------------*/
WITH filtered AS (
    SELECT
        cli_id,
        con_id,
        dt_Rep,
        SECTION_NAME,
        TSEGMENTNAME,
        PROD_NAME_RES,
        OUT_RUB
    FROM  ALM.ALM.Balance_Rest_All WITH (NOLOCK)
    WHERE dt_Rep IN ( '2024-01-31','2024-02-29','2024-03-31','2024-04-30',
                      '2024-05-31','2024-06-30','2024-07-31','2024-08-31',
                      '2024-09-30','2024-10-31','2024-11-30','2024-12-31',
                      '2025-01-31','2025-02-28','2025-03-31','2025-04-30','2025-05-20')
      AND section_name  IN ('Срочные','До востребования','Накопительный счёт')
      AND cur           = '810'
      AND od_flag       = 1
      AND is_floatrate  = 0
      AND ACC_ROLE      = 'LIAB'
      AND AP            = 'Пассив'
      AND TSEGMENTNAME IN ('ДЧБО','Розничный бизнес')
      AND BLOCK_NAME    = 'Привлечение ФЛ'
      AND prod_name_res IS NOT NULL
      AND out_rub       IS NOT NULL
      AND prod_name_res IN (SELECT prod_name_res FROM @Products)   -- << ваши вклады
)

/* ------------------------------------------------------------------
   2. Для каждого cli_id находим первую дату появления → вычисляем квартал
-------------------------------------------------------------------*/
, first_touch AS (
    SELECT
        cli_id,
        MIN(dt_Rep) AS first_dt_rep
    FROM filtered
    GROUP BY cli_id
)
, vintage AS (
    SELECT
        cli_id,
        CONCAT(DATEPART(year , first_dt_rep), 'Q', DATEPART(quarter, first_dt_rep)) AS vintage_qtr
    FROM first_touch
)

/* ------------------------------------------------------------------
   3. Соединяем баланс с таблицей vintage и считаем агрегаты
-------------------------------------------------------------------*/
SELECT
    f.dt_Rep,                         -- срез портфеля (месяц)
    v.vintage_qtr,                    -- квартал «первого появления» клиента
    f.SECTION_NAME,
    f.TSEGMENTNAME,
    f.PROD_NAME_RES,
    SUM(f.OUT_RUB)           AS sum_OUT_RUB,   -- объём портфеля
    COUNT(DISTINCT f.cli_id) AS cnt_clients,   -- число уникальных клиентов
    COUNT(DISTINCT f.con_id) AS cnt_con        -- число договоров (как было)
FROM        filtered f
INNER JOIN  vintage  v  ON v.cli_id = f.cli_id
GROUP BY
    f.dt_Rep,
    v.vintage_qtr,
    f.SECTION_NAME,
    f.TSEGMENTNAME,
    f.PROD_NAME_RES
ORDER BY
    v.vintage_qtr,       -- удобнее сначала по винтажу
    f.dt_Rep,
    f.SECTION_NAME,
    f.TSEGMENTNAME,
    f.PROD_NAME_RES;
