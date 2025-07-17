Ошибка «недопустимое имя столбца» появляется, когда в
итоговых `SELECT`-ах я ссылался на псевдонимы столбцов, которых нет
в таблице `#roll_match`. Исправляем оба блока, явно указывая
агрегации и избавляясь от лишнего символа «/».

```sql
/*────────── итоги по всему набору ──────────*/
SELECT
        total_deals   = COUNT(*) ,
        covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
        pct_deals     = 100.0 *
                        SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
                        / COUNT(*) ,
        total_rub     = SUM(out_rub) ,
        covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
        pct_rub       = 100.0 *
                        SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
                        / NULLIF(SUM(out_rub),0)
FROM    #roll_match;


/*────────── детализация по BALANCE_GROUP × TERM_GROUP ──────────*/
SELECT
        BALANCE_GROUP ,
        TERM_GROUP    ,
        deals_tot = COUNT(*) ,
        deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
        pct_deals = 100.0 *
                    SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
                    / COUNT(*) ,
        rub_tot   = SUM(out_rub) ,
        rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
        pct_rub   = 100.0 *
                    SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
                    / NULLIF(SUM(out_rub),0)
FROM    #roll_match
GROUP  BY BALANCE_GROUP , TERM_GROUP
ORDER BY pct_rub DESC;
```

*Изменения* — только в двух SELECT-ах; остальной скрипт оставьте как есть.
Теперь все используемые столбцы точно присутствуют в `#roll_match`, и
запрос выполнится без ошибок.
