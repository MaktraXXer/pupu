Ниже разберём несколько возможных причин, почему вы могли получить «пустые» названия банков в топ-5 и только суммарное значение в «ОСТАЛЬНЫЕ_БАНКИ», а также приведём рабочий пример запроса без динамического SQL для **T-SQL** (Microsoft SQL Server). 

---

## 1. Убедитесь, что в данных действительно есть **> 0** строк

Банально, если в таблице `sbp_statistic` нет данных или нет отличных от нуля значений в столбцах `INCOMING_SUM_TRANS`, то результат будет пустой (кроме возможно «ОСТАЛЬНЫЕ_БАНКИ»). Проверьте простым запросом:

```sql
SELECT TOP 10 *
FROM sbp_statistic;
```

Убедитесь, что:
1. Строки есть.
2. `INCOMING_BANK_NAME` (или `OUTGOING_BANK_NAME`) **не пустое** и **не NULL**.
3. `INCOMING_SUM_TRANS` (или `OUTGOING_SUM_TRANS`) действительно > 0.

Если все суммы в `INCOMING_SUM_TRANS` нулевые или NULL, то `ROW_NUMBER()` проставится, но потом при отборе `rn <= 5` может что-то выглядеть странно. Однако в любом случае должны выводиться названия банков.

---

## 2. Проверьте правильность **названий полей**

Частая ошибка – опечатка (например, `INCOMMING_...` вместо `INCOMING_...`). Сверьтесь:

- **INCOMING_BANK_NAME**
- **INCOMING_SUM_TRANS**
- **OUTGOING_BANK_NAME**
- **OUTGOING_SUM_TRANS**

---

## 3. Убедитесь, что ваш движок **поддерживает оконные функции** (ROW_NUMBER)

Код ниже – **T-SQL** для Microsoft SQL Server (SQL 2012+).  
Если вы используете **MySQL** до 8.0, в нём оконные функции не поддерживались. Там придётся либо обновиться до 8.0+, либо использовать другие приёмы.

---

## 4. Пример рабочего кода (T-SQL для входящих переводов)

Попробуйте **точно** этот код:

```sql
WITH ranked_in AS (
    SELECT
        WEEK_START,
        WEEK,
        INCOMING_BANK_NAME,
        INCOMING_SUM_TRANS,
        /* Ранжируем банки по сумме входящих переводов в каждой неделе */
        ROW_NUMBER() OVER (
            PARTITION BY WEEK_START, WEEK
            ORDER BY INCOMING_SUM_TRANS DESC
        ) AS rn
    FROM sbp_statistic
    /* Условно можете исключить строки, где нет названия банка или 0 сумм:
       WHERE INCOMING_BANK_NAME IS NOT NULL
         AND INCOMING_SUM_TRANS > 0
    */
)
SELECT
    WEEK_START,
    WEEK,
    INCOMING_BANK_NAME,
    INCOMING_SUM_TRANS
FROM ranked_in
WHERE rn <= 5  -- Топ-5 банков
UNION ALL
SELECT
    WEEK_START,
    WEEK,
    'ОСТАЛЬНЫЕ_БАНКИ' AS INCOMING_BANK_NAME,
    SUM(INCOMING_SUM_TRANS) AS INCOMING_SUM_TRANS
FROM ranked_in
WHERE rn > 5
GROUP BY WEEK_START, WEEK
ORDER BY WEEK_START, WEEK, INCOMING_BANK_NAME; -- Для наглядности
```

Что делается:

1. **ranked_in**:
   - Для каждой `(WEEK_START, WEEK)` упорядочиваем записи (банки) по полю `INCOMING_SUM_TRANS` по убыванию.
   - `ROW_NUMBER()` присваивает уникальный порядковый номер от 1 до ... в пределах каждой недели.
2. Первый `SELECT ... WHERE rn <= 5` – выводим банки, которые заняли места с 1 по 5.
3. `UNION ALL` со вторым `SELECT`, где мы группируемся по `(WEEK_START, WEEK)` и суммируем все банки с `rn > 5` (это и есть «ОСТАЛЬНЫЕ_БАНКИ»).

### Почему могут быть пустые результаты по топ-5?

- Если в какой-то неделе **меньше пяти** банков, то всё равно будет вывод (rn=1,2,...).  
- Если в какой-то неделе вообще **нет данных** (или `INCOMING_SUM_TRANS = 0` у всех), то может быть, что `ROW_NUMBER()` присвоится, но суммы будут 0. Названия банка, однако, должны выводиться.

Если у вас получается, что **INCOMING_BANK_NAME** пустое (пустая строка) – значит в самой таблице в этом поле нет значения или оно `NULL`.  
Чтобы не выводить «пустые» имена, можно подстраховаться:

```sql
COALESCE(INCOMING_BANK_NAME, 'Неизвестный_банк') AS INCOMING_BANK_NAME
```

(тогда вместо NULL будет выводиться «Неизвестный_банк»)

---

## 5. Аналогичный код для **исходящих** переводов

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
)
SELECT
    WEEK_START,
    WEEK,
    OUTGOING_BANK_NAME,
    OUTGOING_SUM_TRANS
FROM ranked_out
WHERE rn <= 5  -- топ-5
UNION ALL
SELECT
    WEEK_START,
    WEEK,
    'ОСТАЛЬНЫЕ_БАНКИ' AS OUTGOING_BANK_NAME,
    SUM(OUTGOING_SUM_TRANS) AS OUTGOING_SUM_TRANS
FROM ranked_out
WHERE rn > 5
GROUP BY WEEK_START, WEEK
ORDER BY WEEK_START, WEEK, OUTGOING_BANK_NAME;
```

---

## 6. Итоги

1. **Проверьте**, что данные реально есть и не все нулевые.
2. **Скопируйте** вышеуказанный запрос (T-SQL) буквально, поправив названия таблицы и полей, если нужно.
3. **Убедитесь**, что работаете на SQL Server 2012+ (или другом движке, поддерживающем оконные функции).
4. Если нужно заменить `NULL`-названия банков, используйте `COALESCE(INCOMING_BANK_NAME, 'Неизвестно')`.

С таким решением без «динамического PIVOT» вы получаете по несколько строк на каждую неделю (топ-5 + одна строка с «ОСТАЛЬНЫЕ_БАНКИ»). Далее в Excel вы сможете построить обычную сводную таблицу: 
- в строки/ось X – недели, 
- в столбцы – **INCOMING_BANK_NAME** (включая «ОСТАЛЬНЫЕ_БАНКИ»), 
- как значение – `INCOMING_SUM_TRANS`. 

Так вы получите нужную визуализацию без динамического SQL.


