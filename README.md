/* ==== 9. СВОД И «ПЕРЕЛИВ» =====================================*/
DROP TABLE IF EXISTS #daily_all;
SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con,prod_id
INTO   #daily_all
FROM (
     SELECT * FROM #fix
     UNION ALL SELECT * FROM #float
     UNION ALL SELECT * FROM #base) q;

/* клиенты, у кого 1-го числа есть promo-FIX */
;WITH first_day AS (
        SELECT DISTINCT cli_id,
               m1 = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
        FROM   #daily_all
        WHERE  prod_id = 654
          AND  dt_rep  = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
)
, best AS (
        /* агрегируем promo-деньги → max(rate_con) */
        SELECT  cli_id,
                dt_rep = m1,
                out_rub = SUM(out_rub),
                rate_con = MAX(rate_con)
        FROM   #daily_all d
        JOIN   first_day f ON f.cli_id=d.cli_id AND f.m1=d.dt_rep
        WHERE  d.prod_id = 654
        GROUP  BY cli_id,m1
)
, union_all AS (
        /* заменяем «рассыпанные» promo одной строкой */
        SELECT NULL con_id, cli_id, NULL TSEGMENTNAME,
               out_rub, dt_rep, rate_con, 654 AS prod_id
        FROM   best
        UNION ALL
        SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con,prod_id
        FROM   #daily_all d
        WHERE  NOT EXISTS (SELECT 1
                           FROM first_day f
                           WHERE f.cli_id = d.cli_id
                             AND f.m1     = DATEFROMPARTS(YEAR(d.dt_rep),MONTH(d.dt_rep),1)
                             AND d.prod_id = 654)
)
/* ==== 10. Итог портфеля и справочники =========================*/
TRUNCATE TABLE WORK.Forecast_BalanceDaily_NS;
INSERT WORK.Forecast_BalanceDaily_NS
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   union_all
GROUP  BY dt_rep;

/* —--- выводим, как просили --- */
SELECT * FROM WORK.NS_Spreads           ORDER BY dt_open,      TSEGMENTNAME;
SELECT * FROM WORK.NS_PromoRates        ORDER BY month_first,  TSEGMENTNAME;
SELECT * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;
