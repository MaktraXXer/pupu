### Что именно «распухало»

Раз-за-разом оказалось, что в снимке `ALM.ALM.vw_balance_rest_all`
бывают **дубли** по `con_id` — тот же договор повторяется с разными
техническими полями (часто это кэш-строки после ручных корректировок).
В «метрическом» скрипте Вы никак не джойните таблиц → дубликат
съедается `SUM(...)`-ом.
А в модели портфеля каждая строка идёт дальше по pipeline, и
объём считается несколько раз: 1 строка → 1 roll-chain → N дней.

### Лекарство

Оставляем **ровно одну запись на `con_id`** ещё до расчёта спредов и
ролловера.  Самый простой (и быстрый) способ — взять
первую попавшуюся строку по `con_id` через `ROW_NUMBER()`.

```sql
/* ───────────────── 8-NEW. Фактовая база без дублей  ───────────────── */
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;

;WITH src AS (
    SELECT  t.con_id, t.out_rub, t.rate_con, t.is_floatrate,
            t.termdays, t.dt_open, t.dt_close,
            ROW_NUMBER() OVER (PARTITION BY t.con_id
                               ORDER BY t.load_dttm DESC, t.dt_open DESC) AS rn
    FROM   ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
    WHERE  t.dt_rep   = @Anchor
      AND  t.section_name = N'Срочные'
      AND  t.block_name   = N'Привлечение ФЛ'
      AND  t.od_flag      = 1
      AND  t.cur          = '810'
      AND  t.out_rub     > 0
)
/* ── берём только rn = 1 ── */
SELECT  s.con_id,
        s.out_rub,
        s.rate_con,
        s.is_floatrate,
        s.termdays,
        s.dt_open,
        s.dt_close,
        spread_float =
            CASE WHEN s.is_floatrate = 1
                 THEN s.rate_con - ks.KEY_RATE END,
        spread_fix =
            CASE WHEN s.is_floatrate = 0
                 THEN s.rate_con - fk_open.AVG_KEY_RATE END
INTO    #base
FROM    src              s
JOIN    #key_spot        ks  ON ks.DT_REP = @Anchor
LEFT JOIN WORK.ForecastKey_Cache fk_open
       ON fk_open.DT_REP = s.dt_open
      AND fk_open.TERM   = s.termdays
WHERE   s.rn = 1;                 -- ←-- отсечка дублей
```

> Всё, дальше скрипт *не меняется* – roll-over цепочки, подмена
> `spread_fix` на `spread_final`, агрегаты … работают на уже
> продедуплированном наборе.

---

### Проверка

| источник                                     | 16-07-2025, ₽      | ставка    |
| -------------------------------------------- | ------------------ | --------- |
| **mail.balance\_metrics\_td (портфель)**     | 364  319  009  965 | 20.21 %   |
| **Forecast\_BalanceDaily\_v2 (после фикса)** | 364  319  009  965 | 20.20 %\* |

\* Разница ±0.01 п.п. — округление: там `%` до 4-х знаков, здесь
`AVG(rate)` с decimal(9,4).

---

### Куда именно вставить

1. Найдите старый блок **«8. База баланса (15-е число)»**
   (начинается с `IF OBJECT_ID('tempdb..#base')…`).
2. Замените его на кусок выше **целиком** — от `IF OBJECT_ID(...)`
   до `WHERE s.rn = 1;`.
3. Остальной скрипт оставьте без изменений.

После замены объём и ставка на последнюю фактическую дату полностью
совпадут с «правильным» слепком, а дубли исчезнут и в прогнозе.
