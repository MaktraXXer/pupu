Для решения задачи необходимо создать два запроса: для анализа входящих и исходящих переводов через СБП. Каждый запрос будет формировать таблицу с динамическими колонками, соответствующими топ-5 банкам за каждую неделю, и суммой переводов для остальных банков.

### Запрос 1: Входящие переводы
```sql
WITH weekly_rank AS (
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
top_banks AS (
  SELECT DISTINCT INCOMING_BANK_NAME
  FROM weekly_rank
  WHERE rank <= 5
)
SELECT 
  wr.WEEK_START,
  wr.WEEK,
  MAX(CASE WHEN wr.INCOMING_BANK_NAME = 'Сбер' AND wr.rank <= 5 THEN wr.total_incoming END) AS "Сбер",
  MAX(CASE WHEN wr.INCOMING_BANK_NAME = 'ВТБ' AND wr.rank <= 5 THEN wr.total_incoming END) AS "ВТБ",
  MAX(CASE WHEN wr.INCOMING_BANK_NAME = 'АЛЬФА-БАНК' AND wr.rank <= 5 THEN wr.total_incoming END) AS "АЛЬФА",
  MAX(CASE WHEN wr.INCOMING_BANK_NAME = 'ГАЗПРОМБАНК' AND wr.rank <= 5 THEN wr.total_incoming END) AS "ГАЗПРОМБАНК",
  MAX(CASE WHEN wr.INCOMING_BANK_NAME = 'ПСБ' AND wr.rank <= 5 THEN wr.total_incoming END) AS "ПСБ",
  MAX(CASE WHEN wr.INCOMING_BANK_NAME = 'РСХБ' AND wr.rank <= 5 THEN wr.total_incoming END) AS "РСХБ",
  SUM(CASE WHEN wr.rank > 5 THEN wr.total_incoming ELSE 0 END) AS "ОСТАЛЬНЫЕ_БАНКИ"
FROM weekly_rank wr
GROUP BY wr.WEEK_START, wr.WEEK
ORDER BY wr.WEEK_START;
```

### Запрос 2: Исходящие переводы
```sql
WITH weekly_rank_out AS (
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
)
SELECT 
  wr.WEEK_START,
  wr.WEEK,
  MAX(CASE WHEN wr.OUTGOING_BANK_NAME = 'Сбер' AND wr.rank <= 5 THEN wr.total_outgoing END) AS "Сбер",
  MAX(CASE WHEN wr.OUTGOING_BANK_NAME = 'ВТБ' AND wr.rank <= 5 THEN wr.total_outgoing END) AS "ВТБ",
  MAX(CASE WHEN wr.OUTGOING_BANK_NAME = 'АЛЬФА-БАНК' AND wr.rank <= 5 THEN wr.total_outgoing END) AS "АЛЬФА",
  MAX(CASE WHEN wr.OUTGOING_BANK_NAME = 'ГАЗПРОМБАНК' AND wr.rank <= 5 THEN wr.total_outgoing END) AS "ГАЗПРОМБАНК",
  MAX(CASE WHEN wr.OUTGOING_BANK_NAME = 'ПСБ' AND wr.rank <= 5 THEN wr.total_outgoing END) AS "ПСБ",
  MAX(CASE WHEN wr.OUTGOING_BANK_NAME = 'РСХБ' AND wr.rank <= 5 THEN wr.total_outgoing END) AS "РСХБ",
  SUM(CASE WHEN wr.rank > 5 THEN wr.total_outgoing ELSE 0 END) AS "ОСТАЛЬНЫЕ_БАНКИ"
FROM weekly_rank_out wr
GROUP BY wr.WEEK_START, wr.WEEK
ORDER BY wr.WEEK_START;
```

### Пояснения:
1. **CTE `weekly_rank`**: Ранжирует банки по сумме переводов за каждую неделю.
2. **CTE `top_banks`**: Определяет уникальные банки, которые когда-либо входили в топ-5.
3. **Основной запрос**: 
   - Для каждой недели проверяет, входит ли банк в топ-5, и выводит сумму переводов.
   - Если банк не в топ-5, значение `NULL`.
   - Сумма переводов для остальных банков объединяется в колонку **ОСТАЛЬНЫЕ_БАНКИ**.

Эти запросы позволяют построить сводные таблицы, удобные для визуализации в Excel, где каждая колонка соответствует банку из топ-5, а строки отражают данные за неделю.
