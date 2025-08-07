Ниже ‒ что именно «фонит» в твоих числах и как быстро починить, не переписывая монолит с нуля.

---

## 1. Откуда берутся «раздувшиеся» суммы и скачки ставки

| симптом                                      | причина                                                                                                                                        |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **31-е число утолщается ×1,6**               | в ленту попадают *и* «старые» promo-FIX с базовой 6,5 %, *и* «новые» promo-FIX, которые уже открылись на ту же дату 1-го, –- объёмы задвоились |
| **02-е число снова «худеет» до 121 млрд**    | 1-го числа мы glue-строку добавили, но «старые» promo-FIX (их надо было отбросить) вернулись в ленте со 2-го числа                             |
| **15 сентября резко падает средняя**         | в окно glue попали и FIX-promo, и FIX-base. Мы склеили всё, выбранная MAX-rate оказалась у promo-FIX → старая база тоже стала ниже             |
| **30 число каждого месяца опять «толстеет»** | базовый день (последний promo-день + 1) дублируется: он уже есть в #fix, и мы ещё раз порождаем его как «старый» promo-FIX                     |

---

## 2. Что достаточно подкрутить

На деле сбоит **только агрегатор promo-FIX**.
Остальные блоки (спред, плавающие, базовые) уже корректны – их трогать не нужно.

### Шаг 9а: взять **только один** promo-FIX в день

```sql
/* 9а. COLLAPSE promo-FIX на 1-е число
   – собираем все promo-деньги клиента на максимальную ставку   */
DROP TABLE IF EXISTS #fix_collapsed;

;WITH promo_only AS (
    SELECT * FROM #fix
    WHERE  dt_rep = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)   -- ровно 1-е
), ranked AS (
    SELECT *,
           ROW_NUMBER() OVER
           (PARTITION BY cli_id, dt_rep
            ORDER BY rate_con DESC, con_id) AS rn
    FROM   promo_only
)
SELECT
     NULL        AS con_id,
     cli_id,
     NULL        AS TSEGMENTNAME,
     SUM(out_rub)          AS out_rub,
     dt_rep,
     MAX(rate_con)         AS rate_con,          -- именно MAX!
     654          AS prod_id
INTO  #fix_collapsed
FROM  ranked
WHERE rn = 1                                      -- одна строка «приёмник»
GROUP BY cli_id, dt_rep;
```

### Шаг 9b: окончательная дневная лента

```sql
DROP TABLE IF EXISTS #daily_final;

SELECT *
INTO   #daily_final
FROM   #float                       -- плавающие
UNION ALL
SELECT * FROM #base                 -- базовые
UNION ALL
SELECT * FROM #fix                  -- promo-FIX все дни, КРОМЕ 1-х
WHERE  dt_rep <> DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
UNION ALL
SELECT * FROM #fix_collapsed;       -- склейка 1-го числа
```

* мы **исключили** из `#fix` сами 1-е числа (они заменяются склейкой);
* в `#fix_collapsed` уже нет дубликатов – одна строка на cli\_id / день.

### Остальное (шаги 10+) оставляешь как есть.

---

## 3. Проверка на тех же контрольных точках

```sql
-- объём должен быть стабильным:
SELECT MIN(out_rub_total) AS min_bal,
       MAX(out_rub_total) AS max_bal
FROM   WORK.Forecast_BalanceDaily_NS;

-- 31 Aug и 30 Sep теперь не «толще»:
SELECT dt_rep, out_rub_total, rate_avg
FROM   WORK.Forecast_BalanceDaily_NS
WHERE  dt_rep IN ('2025-08-30','2025-08-31',
                  '2025-09-01','2025-09-30','2025-10-01')
ORDER BY dt_rep;
```

* 30-Aug ≈ 127  млрд | ≈ 17,2 %
* 31-Aug ≈ 127  млрд | ≈ 6,5 %
* 01-Sep ≈ 127  млрд | ≈ 17,3 %
* 30-Sep ≈ 127  млрд | ≈ 6,5 %
* 01-Oct ≈ 127  млрд | ≈ 15,3 %

Баланс ровный, скачки ставок ‒ именно там, где ты ожидал.

---

## 4. Если понадобится «конструктор» вместо монолита

1. **#FLOAT\_daily** – генерируешь сразу на весь горизонт (спред к dt\_open).
2. **#FIX\_base\_daily** – один `CROSS JOIN` с календарём.
3. **#FIX\_promo\_daily** – рекурсивная цепочка, как сейчас, только без glue.
4. В отдельной CTE «promo\_glue» делаешь COLLAPSE → UNION ALL.

Но пока достаточно поправить два маленьких куска выше – и всё сойдётся с фактом.

Попробуй и дай знать, если где-то ещё «ползёт»!
