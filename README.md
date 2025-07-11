DECLARE @first_month date = '2024-01-01';   -- начало окна
DECLARE @last_month  date = '2025-06-30';   -- конец окна

/* 1. самый поздний dt_rep каждого месяца */
WITH month_last AS (
    SELECT  *,
            ROW_NUMBER() OVER (PARTITION BY FORMAT(dt_rep,'yyyyMM')
                               ORDER BY dt_rep DESC) AS rn
    FROM    ALM_TEST.WORK.VW_FL_DEPO_TERM_WEEK
    WHERE   dt_rep BETWEEN @first_month AND @last_month
      AND   BLOCK_NAME   = 'Привлечение ФЛ'
      AND   SECTION_NAME = 'Срочные'
),

/* 2. «срезы» конца месяца */
snap AS (
    SELECT *
    FROM   month_last
    WHERE  rn = 1
),

/* 3. нормализуем даты */
snap_norm AS (
    SELECT  *,
            TRY_CONVERT(date, generation, 23)           AS generation_dt,     -- 2025-07-31 → 2025-07-31
            EOMONTH(dt_rep)                             AS dt_rep_month_end   -- 2025-07-31 → 2025-07-31
    FROM   snap
)

/* 4. оставляем только «новые» договоры месяца */
SELECT
       dt_rep_month_end  AS dt_rep,     -- 31.01.2024, 29.02.2024, …
       generation,                      -- 2025-07-31
       con_id, acc_id, cur, out_rub,
       SEGMENT_ID, TSEGMENTNAME, termdays,
       attr_REPLENISH, attr_WITHDRAWAL,
       PROD_NAME_RES, rate_con,
       DT_OPEN_fact, dt_close,
       [Срок, дн.], term_name_G,
       [ставка внешняя], [ТС],
       loaddate
FROM   snap_norm
WHERE  EOMONTH(generation_dt) = dt_rep_month_end         -- совпадение месяца
ORDER  BY dt_rep_month_end, con_id;
