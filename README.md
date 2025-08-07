Исправлять надо **только шаг 9** – там, где мы после объединения трёх потоков ставок
(`(#fix ∪ #float ∪ #base)`) «схлопываем» все promo-FIX клиента в одну строку
и добавляем её обратно.  Всё остальное (спреды, расчёт ставок) работает
корректно.

### Что меняем

1. **Заменяем** объявление `#daily_all` и последующий CTE-блок
   `first_day / best / union_all`
2. **Вставляем** вместо него кусок «9а/9b» из сообщения, а именно:

```sql
/* ── 9. конструируем дневную ленту ──────────────────────────── */
DROP TABLE IF EXISTS #daily_all;
SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con,prod_id
INTO   #daily_all
FROM   #fix
UNION ALL
SELECT * FROM #float
UNION ALL
SELECT * FROM #base;

/* ── 9а. убираем дубли promo-FIX и клеим их в одну строку ───── */
DROP TABLE IF EXISTS #daily_mix;
SELECT *
INTO   #daily_mix
FROM   #daily_all
WHERE  NOT (prod_id = 654
        AND dt_rep = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1));

WITH first_fix AS (
     SELECT   cli_id,
              DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1) AS m1
     FROM     #daily_all
     WHERE    prod_id = 654
       AND    dt_rep  = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
)
SELECT  NULL      AS con_id,
        d.cli_id,
        NULL      AS TSEGMENTNAME,
        SUM(d.out_rub)            AS out_rub,
        f.m1       AS dt_rep,
        MAX(d.rate_con)           AS rate_con,
        654        AS prod_id
INTO    #promo_glue
FROM     #daily_all d
JOIN     first_fix  f
     ON  f.cli_id = d.cli_id
    AND  f.m1     = d.dt_rep
WHERE    d.prod_id = 654
GROUP BY d.cli_id, f.m1;

/* ── 9b. итоговая лента для агрегата ────────────────────────── */
DROP TABLE IF EXISTS #daily_final;
SELECT *
INTO   #daily_final
FROM (
      SELECT * FROM #daily_mix
      UNION ALL
      SELECT * FROM #promo_glue
) x;
```

3. **Агрегат (шаг 10)** теперь считаем из `#daily_final`,
   остальной код – без изменений:

```sql
TRUNCATE TABLE WORK.Forecast_BalanceDaily_NS;

INSERT WORK.Forecast_BalanceDaily_NS
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #daily_final
GROUP  BY dt_rep;
```

### Почему это чинит проблему

* `#daily_mix` – чистая лента **без** promo-FIX на 1-е число
  (мы их временно убрали, чтобы не было дублей).
* `#promo_glue` – одна строка на клиента и 1-е число с
  *агрегированным объёмом* и *max-ставкой*.
  В исходном варианте promo-FIX оставались **и как отдельные строки, и как
  «склейка»**, поэтому объём удваивался.
* `#daily_final` = `#daily_mix ∪ #promo_glue` – корректный набор строк:
  объём = фактическому, ставка меняется только там, где должна.

### Проверка

```sql
-- должен держаться постоянный объём до 30-го числа
SELECT TOP (40) * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;

-- 30-е числа: ставка 6.5 %  (base)
-- 01-е:       promo (0.173 / 0.1709 …)
-- 15-е сент:  движение только FLOAT (-2 п.п. ключа)
```

После замены блоков 9-10 объём на 04.08 и далее будет ≈ 127 258 млрд,
а «ступеньки» ставок совпадут с описанной логикой.
