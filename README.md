### почему «недопустимое имя столбца»

1. **`GO` = разрыв пакета.**
   Если после шага 6 поставить `GO`, то `#match` создаётся,
   пакет заканчивается, а в новом пакете (шаг 7) таблица уже
   «другая» — пустой каркас без колонок. Отсюда ошибки.

2. **`SELECT rf.*` без явного списка.**
   Когда в следующем пакете таблица пересоздаётся
   конструкцией `SELECT … INTO #match`, SQL Server
   видит-то лишь два выражения (`rf.*` и `spread_mkt`)
   и на лету называет их `[Expr1000]`, `[Expr1001]`.
   Поэтому позднее обращение к `bucket`, `out_rub` и т.д.
   вызывает «недопустимое имя».

---

## надёжное решение

* **Не делите** скрипт `GO`.
  Вся работа с `#match` должна идти **в одном пакете**.
* Явно укажите список колонок при вставке в `#match`.

Ниже фрагмент «6 + 7» без `GO` и с явным перечислением полей.
Запускайте целиком — ошибки не будет.

```sql
/* ───── 6. каскад сопоставления → #match ───── */
IF OBJECT_ID('tempdb..#match') IS NOT NULL DROP TABLE #match;

SELECT
        rf.con_id,
        rf.out_rub,
        rf.rate_con,
        rf.bucket,
        rf.r,
        rf.TERM_GROUP,
        rf.PROD_NAME_RES,
        rf.TSEGMENTNAME,
        rf.conv,
        rf.conv_norm,
        /* итоговый «рыночный» спред */
        spread_mkt = COALESCE
        (
          /* 6-а. родная conv_norm + тот же или крупнее bucket */
          ( SELECT TOP (1) m.spread_mkt
            FROM   #mkt        m
            JOIN   #bucket_def b_m ON b_m.bucket = m.bucket
            WHERE  m.conv_norm     = rf.conv_norm
              AND  m.TERM_GROUP    = rf.TERM_GROUP
              AND  m.PROD_NAME_RES = rf.PROD_NAME_RES
              AND  m.TSEGMENTNAME  = rf.TSEGMENTNAME
              AND  b_m.r           >= rf.r
            ORDER  BY b_m.r ),
          /* 6-б. агрегат без bucket, только для ДЧБО */
          CASE WHEN rf.TSEGMENTNAME = N'ДЧБО'
               THEN ( SELECT ma.spread_any
                      FROM   #mkt_any ma
                      WHERE  ma.conv_norm     = rf.conv_norm
                        AND  ma.TERM_GROUP    = rf.TERM_GROUP
                        AND  ma.PROD_NAME_RES = rf.PROD_NAME_RES
                        AND  ma.TSEGMENTNAME  = rf.TSEGMENTNAME )
          END
        )
INTO    #match
FROM    #roll_fix rf;

/* ───── 7. отчёты (тот же пакет, без GO!) ───── */

/* 7-а. общая сводка */
SELECT
    total_deals   = COUNT(*) ,
    covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals     = 100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
                    / COUNT(*) ,
    total_rub     = SUM(out_rub) ,
    covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub       = 100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
                    / NULLIF(SUM(out_rub),0)
FROM #match;

/* 7-б. детализация */
SELECT
    bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv,
    deals_tot = COUNT(*) ,
    deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals = 100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
                / COUNT(*) ,
    rub_tot   = SUM(out_rub) ,
    rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub   = 100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
                / NULLIF(SUM(out_rub),0)
FROM #match
GROUP BY bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv
ORDER BY pct_rub DESC;
```

### итоги

* выполняйте шаг 6 и 7 **одним пакетом** (без `GO`);
* колонок у `#match` станет ровно столько, сколько перечислено выше,
  и все они будут доступны в отчётах.
