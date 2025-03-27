Для решения задачи создадим два запроса:  
1) **Топ-5 банков по сальдо (Входящие − Исходящие)**  
2) **Топ-5 банков по сальдо (Исходящие − Входящие)**.

---

### **1. Топ-5 банков по сальдо (Входящие − Исходящие):**
```sql
WITH
weekly_saldo AS (
  SELECT 
    WEEK_START,
    WEEK,
    INCOMING_BANK_NAME AS BANK_NAME,
    -- Считаем сальдо: Входящие - Исходящие
    SUM(INCOMING_SUM_TRANS) - COALESCE(SUM(OUTGOING_SUM_TRANS), 0) AS SALDO_SUM
  FROM sbp_statistic
  GROUP BY WEEK_START, WEEK, INCOMING_BANK_NAME
),
ranked_saldo AS (
  SELECT 
    WEEK_START,
    WEEK,
    BANK_NAME,
    SALDO_SUM,
    ROW_NUMBER() OVER (
      PARTITION BY WEEK_START 
      ORDER BY SALDO_SUM DESC
    ) AS saldo_rank
  FROM weekly_saldo
),
top_banks AS (
  SELECT DISTINCT BANK_NAME
  FROM ranked_saldo
  WHERE saldo_rank <= 5
),
final_data AS (
  SELECT
    rs.WEEK_START,
    rs.WEEK,
    tb.BANK_NAME,
    CASE 
      WHEN rs.saldo_rank <= 5 THEN rs.SALDO_SUM 
      ELSE NULL 
    END AS saldo_value
  FROM ranked_saldo rs
  RIGHT JOIN top_banks tb 
    ON rs.BANK_NAME = tb.BANK_NAME
)
SELECT
  WEEK_START,
  WEEK,
  MAX(CASE WHEN BANK_NAME = 'Банк 1' THEN saldo_value END) AS "Банк 1",
  MAX(CASE WHEN BANK_NAME = 'Банк 2' THEN saldo_value END) AS "Банк 2",
  MAX(CASE WHEN BANK_NAME = 'Банк 3' THEN saldo_value END) AS "Банк 3",
  MAX(CASE WHEN BANK_NAME = 'Банк 4' THEN saldo_value END) AS "Банк 4",
  MAX(CASE WHEN BANK_NAME = 'Банк 5' THEN saldo_value END) AS "Банк 5",
  SUM(CASE WHEN saldo_value IS NULL THEN SALDO_SUM ELSE 0 END) AS "ОСТАЛЬНЫЕ_БАНКИ"
FROM final_data
GROUP BY WEEK_START, WEEK
ORDER BY WEEK_START;
```

---

### **2. Топ-5 банков по сальдо (Исходящие − Входящие):**
Измените только направление расчета сальдо:
```sql
WITH
weekly_saldo AS (
  SELECT 
    WEEK_START,
    WEEK,
    OUTGOING_BANK_NAME AS BANK_NAME,
    -- Считаем сальдо: Исходящие - Входящие
    SUM(OUTGOING_SUM_TRANS) - COALESCE(SUM(INCOMING_SUM_TRANS), 0) AS SALDO_SUM
  FROM sbp_statistic
  GROUP BY WEEK_START, WEEK, OUTGOING_BANK_NAME
),
ranked_saldo AS (
  SELECT 
    WEEK_START,
    WEEK,
    BANK_NAME,
    SALDO_SUM,
    ROW_NUMBER() OVER (
      PARTITION BY WEEK_START 
      ORDER BY SALDO_SUM DESC -- Сортировка по убыванию для наибольших отрицательных значений
    ) AS saldo_rank
  FROM weekly_saldo
)
-- Далее логика аналогична первому запросу
```

---

### **Пояснения:**
1. **CTE `weekly_saldo`**:
   - Объединяет данные по входящим и исходящим переводам для каждого банка и недели.
   - Рассчитывает сальдо (`SALDO_SUM`) как разницу между входящими и исходящими переводами.

2. **CTE `ranked_saldo`**:
   - Ранжирует банки по сальдо для каждой недели.

3. **CTE `top_banks`**:
   - Выбирает банки, которые хотя бы раз попали в топ-5 по сальдо.

4. **Финальный запрос**:
   - Формирует таблицу с динамическими колонками для топ-5 банков и суммой остальных.

---

### **Пример вывода:**
| WEEK_START | WEEK       | Банк 1 | Банк 2 | ... | ОСТАЛЬНЫЕ_БАНКИ |
|------------|------------|--------|--------|-----|------------------|
| 2024-01-01 | 1-7 янв    | 5000   | 3000   | ... | -2000            |
| 2024-01-08 | 8-14 янв   | 4500   | 2800   | ... | -1500            |

- Положительные значения: банк получил больше входящих переводов.
- Отрицательные значения (`ОСТАЛЬНЫЕ_БАНКИ`): суммарное сальдо банков вне топ-5.
