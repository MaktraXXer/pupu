/* 1. берём последний доступный dt_rep каждого месяца */
WITH month_last AS (
    SELECT  *,
            ROW_NUMBER() OVER (PARTITION BY FORMAT(dt_rep,'yyyyMM')
                               ORDER BY dt_rep DESC) AS rn
    FROM    ALM_TEST.WORK.VW_FL_DEPO_TERM_WEEK
    WHERE   dt_rep < '2025-07-01'                       -- «до конца июня»
),

/* 2. оставляем только последние записи месяца */
snap AS (
    SELECT *
    FROM   month_last
    WHERE  rn = 1
),

/* 3. нормализуем поле generation:
      01.<MM.YYYY> → date (1-е число месяца)           */
snap_norm AS (
    SELECT
        *,
        TRY_CONVERT(date,                                  -- 01.07.2024
                    '01.'+REPLACE(LTRIM(RTRIM(generation)),' ',''),
                    104)           AS gen_month_start,
        EOMONTH(dt_rep)            AS dt_rep_month_end
    FROM   snap
)

SELECT
       dt_rep_month_end  AS dt_rep,                       -- 31.07.2024 и т.д.
       generation,
       con_id,
       acc_id,
       cur,
       out_rub,
       SEGMENT_ID,
       TSEGMENTNAME,
       termdays,
       attr_REPLENISH,
       attr_WITHDRAWAL,
       PROD_NAME_RES,
       rate_con,
       DT_OPEN_fact,
       dt_close,
       [Срок, дн.],
       term_name_G,
       [ставка внешняя],
       [ТС],
       loaddate
FROM   snap_norm
WHERE  gen_month_start = DATEFROMPARTS(                   -- 1-е число того же месяца,
                         YEAR(dt_rep_month_end),          -- что и выбранный snapshot
                         MONTH(dt_rep_month_end),1)
ORDER  BY dt_rep_month_end, con_id;
