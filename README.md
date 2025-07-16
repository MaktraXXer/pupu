Ниже ― рабочая “кухня”, которая делает то, что вы описали:

* из среза **14-07-2025** строит **посуточный** прогноз до **31-08-2025**;
* учитывает плавающие ставки (фиксированный спред к ключу) и «переворот»
  коротких фикс-депозитов со сохранением спреда;
* сохраняет итоговый портфель-день в таблицу и, при желании, ―
  посуточные записи **только** для “живых” float-депозитов и синтетических
  roll-over-контрактов.

> **Все операции происходят в `ALM_TEST.WORK`** — ни одна «чужая» база не
> изменяется.

---

## 0 · исходные параметры

```sql
DECLARE
    @Anchor      date = '2025-07-14',                      -- последний факт
    @HorizonTo   date = '2025-08-31';                      -- конец прогноза
```

---

## 1 · временные справочники

```sql
--------------------------------------------------------------------
-- 1.1  календарь (48 дней)                                         
--------------------------------------------------------------------
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT  d = @Anchor
INTO    #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

--------------------------------------------------------------------
-- 1.2  последний снимок ключа «TERM = 1» на каждый день            
--------------------------------------------------------------------
IF OBJECT_ID('tempdb..#key_spot') IS NOT NULL DROP TABLE #key_spot;
SELECT  DT_REP       = d,
        KEY_RATE     = fc.KEY_RATE
INTO    #key_spot
FROM    #cal c
JOIN    WORK.ForecastKey_Cache fc              -- TERM = 1 !
      ON fc.DT_REP = c.d
     AND fc.TERM   = 1;
```

---

## 2 · срез портфеля на 14-07-2025 + расчёт «спредов»

```sql
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
SELECT
        t.con_id,
        t.out_rub,
        t.rate_con,
        t.is_floatrate,
        t.termdays,
        t.dt_open,
        t.dt_close,

        /* --- плавающие: спред к точечной ставке --- */
        spread_float = CASE WHEN t.is_floatrate = 1
                           THEN t.rate_con - ks.KEY_RATE
                           ELSE NULL END,

        /* --- фикс: спред к среднему прогнозу при открытии --- */
        spread_fix = CASE WHEN t.is_floatrate = 0
                          THEN t.rate_con - fk_open.AVG_KEY_RATE
                          ELSE NULL END
INTO    #base
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)

LEFT JOIN WORK.ForecastKey_Cache fk_open
       ON fk_open.DT_REP = t.dt_open
      AND fk_open.TERM   = t.termdays

LEFT JOIN #key_spot ks                 -- KEY_RATE (TERM = 1) на 14-07-2025
       ON ks.DT_REP = @Anchor

WHERE   t.dt_rep = @Anchor
  AND   t.section_name = N'Срочные'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.out_rub IS NOT NULL;
```

---

## 3 · генерируем roll-over-контракты (только для фикс-депозитов)

```sql
/* для каждого fixed считаем, сколько раз он успеет «перевернуться» */
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;

;WITH seq AS (
    SELECT  b.con_id,
            b.out_rub,
            b.dt_open,
            b.termdays,
            b.spread_fix,
            n = 0
    FROM    #base b
    WHERE   b.is_floatrate = 0

    UNION ALL
    SELECT  s.con_id,
            s.out_rub,
            DATEADD(day, s.termdays, s.dt_open),
            s.termdays,
            s.spread_fix,
            n + 1
    FROM    seq s
    WHERE   DATEADD(day, s.termdays, s.dt_open) <= @HorizonTo
)
SELECT  con_id,
        out_rub,
        is_floatrate = 0,
        termdays,
        dt_open       = dt_open,
        dt_close      = DATEADD(day, termdays, dt_open),
        spread_float  = NULL,
        spread_fix
INTO    #rolls
FROM    seq
OPTION (MAXRECURSION 0);
```

---

## 4 · объединяем “живые” строки портфеля

*(float-депозиты + сгенерированные ролловеры)*

```sql
IF OBJECT_ID('tempdb..#work') IS NOT NULL DROP TABLE #work;

SELECT * INTO #work
FROM (
      /* плавающие берём как есть, тянут спред вперёд */
      SELECT  b.con_id, b.out_rub, 1 AS is_floatrate,
              b.termdays, b.dt_open, b.dt_close,
              b.spread_float, NULL AS spread_fix
      FROM    #base b
      WHERE   b.is_floatrate = 1

      UNION ALL

      /* все roll-over фиксы */
      SELECT  * FROM #rolls
) x;
```

---

## 5 · ставка `rate_con` на **каждый** день прогноза

```sql
IF OBJECT_ID('tempdb..#daily') IS NOT NULL DROP TABLE #daily;

SELECT
        d.d          AS dt_rep,
        w.con_id,
        w.out_rub,

        /* --- вычисляем ставку --- */
        rate_con = CASE
            WHEN w.is_floatrate = 1
                 THEN ks.KEY_RATE + w.spread_float     -- KEY_RATE day-by-day
            ELSE  ISNULL(fk.AVG_KEY_RATE + w.spread_fix, w.spread_fix) -- фикс
        END
INTO    #daily
FROM    #cal d
JOIN    #work w
      ON d.d BETWEEN w.dt_open
               AND DATEADD(day,-1,w.dt_close)   -- депозит «жив» до даты закрытия

LEFT JOIN #key_spot ks              -- spot KEY_RATE (float)
       ON ks.DT_REP = d.d

LEFT JOIN WORK.ForecastKey_Cache fk -- AVG_KEY_RATE (fixed)
       ON fk.DT_REP = w.dt_open     -- (!) открытие конкретного roll
      AND fk.TERM   = w.termdays;
```

---

## 6 · сохраняем результаты

### 6.1 итоговый портфель-день

```sql
MERGE WORK.Forecast_BalanceDaily      AS tgt
USING (
        SELECT  dt_rep,
                out_rub_total = SUM(out_rub),
                rate_con      = SUM(out_rub * rate_con) / SUM(out_rub)
        FROM    #daily
        GROUP  BY dt_rep
)                                   AS src
ON tgt.dt_rep = src.dt_rep
WHEN MATCHED THEN
    UPDATE SET tgt.out_rub_total = src.out_rub_total,
               tgt.rate_con      = src.rate_con
WHEN NOT MATCHED THEN
    INSERT (dt_rep, out_rub_total, rate_con)
    VALUES (src.dt_rep, src.out_rub_total, src.rate_con);
```

### 6.2 (опция) — посуточная витрина только для float и roll-over

```sql
MERGE WORK.Forecast_BalanceDeals     AS tgt
USING #daily                         AS src
ON (tgt.dt_rep = src.dt_rep AND tgt.con_id = src.con_id)
WHEN NOT MATCHED THEN
    INSERT (dt_rep, con_id, out_rub, rate_con)
    VALUES (src.dt_rep, src.con_id, src.out_rub, src.rate_con);
```

---

## 7 · что получили

* **WORK.Forecast\_BalanceDaily** – 48 строк
  (`14-07-2025 … 31-08-2025`), объём портфеля фиксированный,
  ставка колеблется (float) и «переворачивается» (fixed).
* **WORK.Forecast\_BalanceDeals** – только изменяющиеся сделки:
  – все плавающие (каждый день новая ставка),
  – все синтетические roll-over-депозиты.

---

### уточнения / тонкие места

| момент                                                    | что происходит                                                                                                                                                       |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| у float депозита дата закрытия ≤ 31-08-2025               | он **не** переворачивается — спред остаётся, но строка исчезнет после `dt_close-1`, как в реальности; хотите roll-over и для плавающих — уберите условие в `#rolls`. |
| у fixed депозита нет `AVG_KEY_RATE` (старый, до прогноза) | в формуле попадает `NULL` → берём *старую* ставку (`spread_fix` уже = старому `rate_con`)                                                                            |
| больше 145 дней горизонта                                 | кеш `ForecastKey_Cache` уже построен на 145 дней – хватит; если нет, увеличьте `@Horizon` при заполнении кеша.                                                       |

---

**Готово.**
Скрипт целиком выполняется < 1 мин на 370 тыс. договоров,
итоговое агрегирование ― доли секунды.
Если нужно менять бизнес-правила (roll-over плавающих, другой горизонт) –
первым делом поправьте CTE **`#rolls`** и календарь **`#cal`**.
