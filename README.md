Попробуйте такую версию запроса, где мы сначала вычисляем ранжирование по неделям, затем формируем два набора – для топ‑5 банков и для остальных – и объединяем их через UNION ALL. В результате для каждой недели вы получите по одной строке на каждый банк из топ‑5 и одну строку с агрегированными суммами для остальных банков.

### Входящие переводы

```sql
WITH ranked_in AS (
    SELECT
        WEEK_START,
        WEEK,
        INCOMING_BANK_NAME,
        INCOMING_SUM_TRANS,
        ROW_NUMBER() OVER (
            PARTITION BY WEEK_START, WEEK
            ORDER BY INCOMING_SUM_TRANS DESC
        ) AS rn
    FROM sbp_statistic
),
Top5 AS (
    SELECT
        WEEK_START,
        WEEK,
        INCOMING_BANK_NAME,
        INCOMING_SUM_TRANS
    FROM ranked_in
    WHERE rn <= 5
),
Others AS (
    SELECT
        WEEK_START,
        WEEK,
        'ОСТАЛЬНЫЕ_БАНКИ' AS INCOMING_BANK_NAME,
        SUM(INCOMING_SUM_TRANS) AS INCOMING_SUM_TRANS
    FROM ranked_in
    WHERE rn > 5
    GROUP BY WEEK_START, WEEK
)
SELECT *
FROM Top5
UNION ALL
SELECT *
FROM Others
ORDER BY WEEK_START, WEEK;
```

### Исходящие переводы

```sql
WITH ranked_out AS (
    SELECT
        WEEK_START,
        WEEK,
        OUTGOING_BANK_NAME,
        OUTGOING_SUM_TRANS,
        ROW_NUMBER() OVER (
            PARTITION BY WEEK_START, WEEK
            ORDER BY OUTGOING_SUM_TRANS DESC
        ) AS rn
    FROM sbp_statistic
),
Top5 AS (
    SELECT
        WEEK_START,
        WEEK,
        OUTGOING_BANK_NAME,
        OUTGOING_SUM_TRANS
    FROM ranked_out
    WHERE rn <= 5
),
Others AS (
    SELECT
        WEEK_START,
        WEEK,
        'ОСТАЛЬНЫЕ_БАНКИ' AS OUTGOING_BANK_NAME,
        SUM(OUTGOING_SUM_TRANS) AS OUTGOING_SUM_TRANS
    FROM ranked_out
    WHERE rn > 5
    GROUP BY WEEK_START, WEEK
)
SELECT *
FROM Top5
UNION ALL
SELECT *
FROM Others
ORDER BY WEEK_START, WEEK;
```

**Пояснения:**

- В CTE `ranked_in` (или `ranked_out`) для каждой недели банки ранжируются по сумме переводов (в порядке убывания).  
- CTE `Top5` выбирает строки, где ранг ≤ 5.  
- CTE `Others` агрегирует суммы по всем записям с рангом > 5 для каждой недели и присваивает метку «ОСТАЛЬНЫЕ_БАНКИ».  
- Итоговый запрос объединяет оба набора данных.  

Если в результате у вас для строк с топ‑5 банков выводятся пустые значения, проверьте, что в исходной таблице поле имени банка (INCOMING_BANK_NAME или OUTGOING_BANK_NAME) действительно содержит данные для соответствующих записей. Также убедитесь, что в каждой неделе присутствует достаточно строк, чтобы сформировать топ‑5.
