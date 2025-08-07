/* ── 9а. убираем дубли promo-FIX и клеим их в одну строку ───── */
DROP TABLE IF EXISTS #daily_mix;
SELECT  con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con,prod_id
INTO    #daily_mix
FROM    #daily_all
WHERE  NOT (prod_id = 654
        AND dt_rep = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1));

/* promo_glue в нужном порядке столбцов! */
DROP TABLE IF EXISTS #promo_glue;
SELECT  NULL      AS con_id,
        d.cli_id ,
        NULL      AS TSEGMENTNAME,
        SUM(d.out_rub)            AS out_rub,
        f.m1                      AS dt_rep,
        MAX(d.rate_con)           AS rate_con,
        654       AS prod_id
INTO    #promo_glue
FROM    #daily_all d
JOIN    ( SELECT DISTINCT cli_id ,
                 DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1) AS m1
          FROM   #daily_all
          WHERE  prod_id=654
            AND  dt_rep = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
        ) f
      ON f.cli_id = d.cli_id
     AND f.m1     = d.dt_rep
WHERE   d.prod_id = 654
GROUP BY d.cli_id , f.m1 ;

/* ── 9b. итоговая лента для агрегата ────────────────────────── */
DROP TABLE IF EXISTS #daily_final;
SELECT  con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con,prod_id
INTO    #daily_final
FROM    #daily_mix
UNION ALL
SELECT  con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con,prod_id
FROM    #promo_glue;
