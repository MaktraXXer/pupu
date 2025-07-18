Понял — для дальнейших шагов (проверка конвенции, TSEGMENTNAME и т. п.) эти поля должны попасть в `#base`, а значит и ранее, в `#roll_fix / #match`, они тоже должны присутствовать.

Ниже минимальный патч (два места), который **добавляет**:

* `conv` → `conv_raw`  (оригинальная конвенция из баланса);
* `TSEGMENTNAME`.

### 1. Шаг «roll-fix» — достаём поля из баланса

```sql
/*──── 4. Roll-over фиксы + факт-спред  ────*/
IF OBJECT_ID('tempdb..#roll_fix') IS NOT NULL DROP TABLE #roll_fix;
SELECT  r.con_id,
        r.out_rub,
        b.bucket,
        b.r,
        tg.TERM_GROUP,
        r.PROD_NAME_RES,
        r.TSEGMENTNAME,              -- ← добавили
        conv_raw = r.conv,            -- ← сохраните «как есть»
        conv_norm = CASE WHEN r.conv = 'AT_THE_END' THEN '1M'
                         ELSE CAST(r.conv AS varchar(50)) END,
        spread_fix = r.rate_con - fk_open.AVG_KEY_RATE
INTO    #roll_fix
…
```

> Остальная часть запроса без изменений.

### 2. Шаг «base» — передаём поля в фактовую базу

```sql
/*──── 8. База баланса (16-е число) ────*/
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
SELECT  t.con_id,
        t.out_rub,
        t.rate_con,
        t.is_floatrate,
        t.termdays,
        t.dt_open,
        t.dt_close,
        t.TSEGMENTNAME,                 -- ← добавили
        conv_raw   = t.conv,            -- ← если понадобится
        spread_float = CASE WHEN t.is_floatrate = 1
                                 THEN t.rate_con - ks.KEY_RATE END,
        spread_fix   = CASE WHEN t.is_floatrate = 0
                                 THEN COALESCE(fs.spread_final,
                                               t.rate_con - fk_open.AVG_KEY_RATE) END
INTO    #base
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
JOIN    #key_spot ks  ON ks.DT_REP = @Anchor
LEFT    JOIN WORK.ForecastKey_Cache fk_open
           ON fk_open.DT_REP = t.dt_open
          AND fk_open.TERM   = t.termdays
LEFT    JOIN #fix_spread fs ON fs.con_id = t.con_id
WHERE   t.dt_rep       = @Anchor
  AND   t.section_name = N'Срочные'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.out_rub      IS NOT NULL;
```

---

### Что ещё менять?

* В остальных шагах (`#match`, `#rolls`, расчёт дневных ставок) поля автоматически «протекут» дальше, если они вам нужны — просто обращайтесь к `TSEGMENTNAME` или `conv_raw`.
* Логика расчёта ставок/спредов не тронута, объёмы и средневзвешенная ставка на 2025-07-16 останутся корректными.

Вставьте оба фрагмента вместо соответствующих блоков в текущем скрипте v2 — этого достаточно, чтобы информация о конвенции и сегменте была доступна на всех этапах.
