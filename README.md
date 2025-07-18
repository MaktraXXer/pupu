### Причина расхождения 20.21 % → 19.92 %

Мы «заменили» `spread_fix` на подобранный `spread_final` **уже в первой (n = 0) записи сделки**.
На 16-е июля эта запись должна оставаться фактической (ставка = `rate_con`).
Иначе средневзвешенная ставка факта смещается вниз.

### Что меняем

Исправляем только блок **9. Roll-over с применением spread\_final**
— оставляем факт-спред для `n = 0`, а `spread_final` применяем со второй записи
(при перевороте).

```sql
/*───────────────────── 9. Roll-over с применением spread_final */
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;

;WITH seq AS (
    SELECT con_id, out_rub, is_floatrate, termdays,
           spread_float,               -- всегда факт
           spread_fix,                 -- пока факт
           dt_open,
           n = 0                       -- ← первая (фактовая) запись
    FROM   #base
    UNION ALL
    SELECT s.con_id, s.out_rub, s.is_floatrate, s.termdays,
           s.spread_float,
           s.spread_fix,               -- пока старый, заменим ниже
           DATEADD(day, s.termdays, s.dt_open),
           n + 1
    FROM   seq s
    WHERE  DATEADD(day, s.termdays, s.dt_open) <= @HorizonTo
)
SELECT s.con_id,
       s.out_rub,
       s.is_floatrate,
       s.termdays,
       s.dt_open,
       dt_close = DATEADD(day, s.termdays, s.dt_open),
       s.spread_float,
       /* для n=0 оставляем факт-спред;
          начиная с n=1 — подставляем найденный spread_final */
       spread_fix = CASE
                      WHEN s.n = 0
                           THEN s.spread_fix
                      ELSE ISNULL(fs.spread_final, s.spread_fix)
                    END
INTO   #rolls
FROM   seq         AS s
LEFT   JOIN #fix_spread fs ON fs.con_id = s.con_id
OPTION (MAXRECURSION 0);
```

> *Больше ничего в скрипте трогать не нужно.*

---

#### Итог

* **Объёмы** на 16-е июля остаются правильными (как и после предыдущего патча).
* **Ставка** на 16-е июля теперь рассчитывается из фактических `rate_con`
  и снова даст **20.21 %** вместо 19.92 %.
* Начиная с первого переворота (n = 1) каждый фикс-депозит
  использует подобранный `spread_final` — именно так, как задумывалось.
