, SUM(CASE WHEN t.rate_con IS NOT NULL THEN t.rate_con * t.out_rub ELSE 0 END)
    / NULLIF(SUM(t.out_rub),0)                         AS rate_con
