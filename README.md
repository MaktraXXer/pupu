Ниже ‒ готовые фрагменты T-SQL, которые закрывают все четыре пункта.

---

## 1. Добавляем колонку «остаточная срочность» в витрину

```sql
ALTER TABLE [WORK].[GroupDepositInterestsRate]
    ADD [RESIDUAL_MATUR] DECIMAL(38,6) NULL;   -- дни до DT_CLOSE на дату расчёта
```

> Колонка создаётся c NULL-значениями, поэтому существующие записи автоматически «заполнены» `NULL` – дополнительный `UPDATE` не нужен.

---

## 2 – 4. Изменяем процедуру **\[WORK].\[prc\_GenerateGroupDepositInterestsRateVer1]**

### 2.1 – рассчитываем остаточную срочность на уровне сделок

В первом CTE/темп-таблице `#depSpreads` сразу после блока, где уже рассчитываются `BALANCE_RUB`, `MATUR`, … вставьте строку:

```sql
        ,DATEDIFF(
              DAY
             ,CASE                                     -- дата разреза (как в calc Date)
                  WHEN i.[i] IN (1,4,6) THEN cal.[Date]
                  WHEN i.[i] = 2       THEN DATEADD(DAY,-(DATEPART(WEEKDAY,dep.[DT_OPEN]) - 1), dep.[DT_OPEN])
                  WHEN i.[i] = 3       THEN DATEADD(DAY,-(DATEPART(WEEKDAY,dep.[DT_CLOSE])- 1), dep.[DT_CLOSE])
                  WHEN i.[i] = 5       THEN DATEADD(MONTH, DATEDIFF(MONTH,0,dep.[DT_OPEN]),0)
              END
             ,dep.[DT_CLOSE]
        )                                               [RESIDUAL_MATUR]   -- ← Новое поле
```

*(Если хотите отсечь случаи, когда дата разреза > DT\_CLOSE, оберните `DATEDIFF` во `CASE WHEN … < 0 THEN 0 END`.)*

### 2.2 – агрегируем как средневзвешенное

В «кубовом» объединении `#depSpreads2` добавьте колонку-агрегат рядом с остальными средневзвешенными показателями:

```sql
        ,SUM([BALANCE_RUB] * [RESIDUAL_MATUR]) / SUM([BALANCE_RUB])  [RESIDUAL_MATUR]
```

### 2.3 – пропускаем поле через `#t_for_transf`

Колонка уже идёт в `SELECT *`, ничего менять не нужно.

### 2.4 – записываем в витрину

В `INSERT INTO [ALM_TEST].[WORK].[GroupDepositInterestsRate] …`
добавьте поле в обе части списка:

```sql
    , [RESIDUAL_MATUR]
```

Полный фрагмент «шапки» вставки станет:

```sql
INSERT INTO [ALM_TEST].[WORK].[GroupDepositInterestsRate]
SELECT  [Date]
       ,[TYPE]
       ,CLI_SUBTYPE
       ,MARGIN_TYPE
       ,TERM_GROUP
       ,IS_OPTION
       ,SEG_NAME
       ,TSEGMENTNAME
       ,Spread_KeyRate
       ,BALANCE_RUB
       ,[MATUR]
       ,[RESIDUAL_MATUR]          -- ← новое
       ,[ФОР]
       ,[ССВ]
       ,[ALM_OptionRate]
       ,[ALM_Надбавка]
       ,[MonthlyCONV_OIS]
       ,[MonthlyCONV_TransfertRate]
       ,[MonthlyCONV_ALM_TransfertRate]
       ,[MonthlyCONV_TransfertRate_MOD]
       ,[MonthlyCONV_KBD]
       ,[MonthlyCONV_ForecastKeyRate]
       ,[MonthlyCONV_Rate]
       ,GETDATE()
FROM   #t_for_transf …
```

---

### Итог

* Таблица расширена на колонку `[RESIDUAL_MATUR]` (дни).
* Процедура для каждой сделки считает `DT_CLOSE – Date`, после чего выводит средневзвешенное значение в витрину вместе с остальными показателями.

Скрипт можно накатывать в боевую базу; откат не потребуется, так как изменения обратимы обычным `ALTER TABLE … DROP COLUMN` и возвратом старой версии процедуры.
