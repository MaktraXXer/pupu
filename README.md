CAST(SUM(CASE WHEN rc.rate_use IS NOT NULL THEN rc.rate_use * rc.out_rub END) 
     / NULLIF(SUM(CASE WHEN rc.rate_use IS NOT NULL THEN rc.out_rub END),0) AS decimal(9,4)) AS rate_con
