COALESCE(SUM(CASE WHEN section_name = N'Срочные'           THEN out_rub END),0)/1e6 AS term_rub,
COALESCE(SUM(CASE WHEN section_name = N'До востребования'  THEN out_rub END),0)/1e6 AS dvs_rub,
COALESCE(SUM(CASE WHEN section_name = N'Накопительный счёт' THEN out_rub END),0)/1e6 AS ns_rub,
