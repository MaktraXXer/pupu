Для решения задачи необходимо динамически определять топ-5 банков **для каждой недели** и формировать столбцы на основе банков, которые **хотя бы раз попадали в топ-5** за любой период. Вот запросы для входящих и исходящих переводов:

---

### **Запрос 1: Входящие переводы**
```sql
WITH
-- Ранжируем банки по сумме входящих переводов для каждой недели
weekly_rank_in AS (
  SELECT
    WEEK_START,
    WEEK,
    INCOMING_BANK_NAME,
    SUM(INCOMING_SUM_TRANS) AS total_incoming,
    ROW_NUMBER() OVER (
      PARTITION BY WEEK_START 
      ORDER BY SUM(INCOMING_SUM_TRANS) DESC
    ) AS rank
  FROM sbp_statistic
  GROUP BY WEEK_START, WEEK, INCOMING_BANK_NAME
),
-- Собираем все банки, которые когда-либо были в топ-5 за любую неделю
top_banks_in AS (
  SELECT DISTINCT INCOMING_BANK_NAME
  FROM weekly_rank_in
  WHERE rank <= 5
),
-- Формируем финальную таблицу с динамическими колонками
final_data_in AS (
  SELECT
    wr.WEEK_START,
    wr.WEEK,
    -- Для каждого банка из top_banks_in проверяем, входит ли он в топ-5 на текущей неделе
    MAX(CASE WHEN tb.INCOMING_BANK_NAME = wr.INCOMING_BANK_NAME AND wr.rank <= 5 THEN wr.total_incoming END) AS bank_value,
    -- Сумма для остальных банков
    SUM(CASE WHEN wr.rank > 5 THEN wr.total_incoming ELSE 0 END) AS ОСТАЛЬНЫЕ_БАНКИ
  FROM weekly_rank_in wr
  CROSS JOIN top_banks_in tb
  GROUP BY wr.WEEK_START, wr.WEEK, tb.INCOMING_BANK_NAME
)
-- Преобразуем строки в колонки (PIVOT)
SELECT
  WEEK_START,
  WEEK,
  COALESCE(MAX(CASE WHEN INCOMING_BANK_NAME = 'Сбер' THEN bank_value END), 0) AS "Сбер",
  COALESCE(MAX(CASE WHEN INCOMING_BANK_NAME = 'ВТБ' THEN bank_value END), 0) AS "ВТБ",
  -- ... аналогично для других банков из top_banks_in
  SUM(ОСТАЛЬНЫЕ_БАНКИ) AS "ОСТАЛЬНЫЕ_БАНКИ"
FROM final_data_in
GROUP BY WEEK_START, WEEK
ORDER BY WEEK_START;
```

---

### **Запрос 2: Исходящие переводы**
Аналогичный подход для исходящих переводов:
```sql
WITH
weekly_rank_out AS (
  SELECT
    WEEK_START,
    WEEK,
    OUTGOING_BANK_NAME,
    SUM(OUTGOING_SUM_TRANS) AS total_outgoing,
    ROW_NUMBER() OVER (
      PARTITION BY WEEK_START 
      ORDER BY SUM(OUTGOING_SUM_TRANS) DESC
    ) AS rank
  FROM sbp_statistic
  GROUP BY WEEK_START, WEEK, OUTGOING_BANK_NAME
),
top_banks_out AS (
  SELECT DISTINCT OUTGOING_BANK_NAME
  FROM weekly_rank_out
  WHERE rank <= 5
),
final_data_out AS (
  SELECT
    wr.WEEK_START,
    wr.WEEK,
    MAX(CASE WHEN tb.OUTGOING_BANK_NAME = wr.OUTGOING_BANK_NAME AND wr.rank <= 5 THEN wr.total_outgoing END) AS bank_value,
    SUM(CASE WHEN wr.rank > 5 THEN wr.total_outgoing ELSE 0 END) AS ОСТАЛЬНЫЕ_БАНКИ
  FROM weekly_rank_out wr
  CROSS JOIN top_banks_out tb
  GROUP BY wr.WEEK_START, wr.WEEK, tb.OUTGOING_BANK_NAME
)
SELECT
  WEEK_START,
  WEEK,
  COALESCE(MAX(CASE WHEN OUTGOING_BANK_NAME = 'Сбер' THEN bank_value END), 0) AS "Сбер",
  COALESCE(MAX(CASE WHEN OUTGOING_BANK_NAME = 'ВТБ' THEN bank_value END), 0) AS "ВТБ",
  -- ... аналогично для других банков
  SUM(ОСТАЛЬНЫЕ_БАНКИ) AS "ОСТАЛЬНЫЕ_БАНКИ"
FROM final_data_out
GROUP BY WEEK_START, WEEK
ORDER BY WEEK_START;
```

---

### **Пояснение:**
1. **CTE `weekly_rank_*`**:
   - Ранжирует банки по сумме переводов за каждую неделю.
   - Для входящих переводов используется `INCOMING_BANK_NAME`, для исходящих — `OUTGOING_BANK_NAME`.

2. **CTE `top_banks_*`**:
   - Собирает все банки, которые хотя бы раз попали в топ-5 за любой период.

3. **CTE `final_data_*`**:
   - Для каждой недели и каждого банка из `top_banks_*` проверяет, входит ли он в топ-5 на текущей неделе.
   - Если да — возвращает сумму переводов, если нет — `NULL`.
   - Суммирует переводы для банков вне топ-5 в колонку **ОСТАЛЬНЫЕ_БАНКИ**.

4. **Финальный запрос**:
   - Преобразует строки в колонки с помощью `CASE` и `MAX`.
   - Значения `NULL` заменяются на `0` для удобства визуализации.

---

### **Важно:**
- Если названия банков заранее неизвестны, запросы потребуют модификации (например, использование динамического SQL или скриптов).
- В примере показаны банки "Сбер", "ВТБ" и т.д. — их нужно заменить на реальные названия из ваших данных.




Можно сделать без динамического SQL, но итоговый результат получится в виде набора строк, а не столбцов для каждого банка. То есть, для каждой недели вы получите несколько строк: по одной для каждого банка, оказавшегося в топ-5, и одну строку с агрегатом по «ОСТАЛЬНЫЕ_БАНКИ». Далее, уже в Excel можно построить сводную таблицу или диаграмму, которая распределит данные по столбцам.

Ниже пример для входящих переводов:

```sql
WITH ranked_in AS (
    SELECT
        WEEK_START,
        WEEK,
        INCOMING_BANK_NAME,
        INCOMING_SUM_TRANS,
        ROW_NUMBER() OVER (PARTITION BY WEEK_START, WEEK ORDER BY INCOMING_SUM_TRANS DESC) AS rn
    FROM sbp_statistic
)
-- Выбираем строки для банков, попавших в топ-5, и дополнительную строку для остальных банков
SELECT
    WEEK_START,
    WEEK,
    INCOMING_BANK_NAME,
    INCOMING_SUM_TRANS
FROM ranked_in
WHERE rn <= 5

UNION ALL

SELECT
    WEEK_START,
    WEEK,
    'ОСТАЛЬНЫЕ_БАНКИ' AS INCOMING_BANK_NAME,
    SUM(INCOMING_SUM_TRANS) AS INCOMING_SUM_TRANS
FROM ranked_in
WHERE rn > 5
GROUP BY WEEK_START, WEEK
ORDER BY WEEK_START, WEEK;
```

Аналогично для исходящих переводов:

```sql
WITH ranked_out AS (
    SELECT
        WEEK_START,
        WEEK,
        OUTGOING_BANK_NAME,
        OUTGOING_SUM_TRANS,
        ROW_NUMBER() OVER (PARTITION BY WEEK_START, WEEK ORDER BY OUTGOING_SUM_TRANS DESC) AS rn
    FROM sbp_statistic
)
SELECT
    WEEK_START,
    WEEK,
    OUTGOING_BANK_NAME,
    OUTGOING_SUM_TRANS
FROM ranked_out
WHERE rn <= 5

UNION ALL

SELECT
    WEEK_START,
    WEEK,
    'ОСТАЛЬНЫЕ_БАНКИ' AS OUTGOING_BANK_NAME,
    SUM(OUTGOING_SUM_TRANS) AS OUTGOING_SUM_TRANS
FROM ranked_out
WHERE rn > 5
GROUP BY WEEK_START, WEEK
ORDER BY WEEK_START, WEEK;
```

**Пояснения:**

- В CTE (например, *ranked_in*) для каждой недели с помощью `ROW_NUMBER()` банки ранжируются по сумме переводов.
- В первом запросе выбираются те банки, у которых ранг ≤ 5 (то есть топ-5).
- Во втором запросе для каждой недели суммируются переводы банков, оказавшихся с рангом > 5, и выводится строка с меткой «ОСТАЛЬНЫЕ_БАНКИ».
- Итоговый результат – набор строк, где для каждой недели будет по одной строке на банк из топ-5 и одна строка для остальных.

Если вам важно видеть именно одну строку на неделю с колонками для каждого банка, то без динамического SQL это сделать нельзя, если набор банков не фиксирован. Но подобный формат можно легко получить уже средствами Excel (сводная таблица) после экспорта результата.

Таким образом, без динамического SQL вы можете сначала «пробежаться» по неделям, найти топ-5 банков и сохранить результат в виде строк, а затем уже строить нужное представление данных в Excel.
