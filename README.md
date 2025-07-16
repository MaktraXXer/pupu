### Как извлечь оба средних прогнозных значения одним запросом

**Исходные данные**

| Переменная   | Что это                                                                     | Как получаем                      |
| ------------ | --------------------------------------------------------------------------- | --------------------------------- |
| `@OpenDate`  | дата открытия «старого» депозита                                            | приходит во входных данных модели |
| `@Term`      | срочность депозита в днях                                                   | приходит во входных данных модели |
| `@CloseDate` | плановая дата закрытия старого депозита (а значит — дата открытия «нового») | `DATEADD(day,@Term,@OpenDate)`    |
| `@LastRep`   | самая свежая на сегодня выгрузка прогноза ключа                             | `SELECT MAX(DT_REP)…` из витрины  |

**Что нужно посчитать**

1. `AvgRateAtOpen` — средний прогнозный ключ, который был **известен в день @OpenDate**
   → просто берём `AVG_KEY_RATE` из строки `DT_REP = @OpenDate AND Term = @Term`.

2. `AvgRateAtClose` — средний прогнозный ключ для депозита той же срочности, **если он откроется в @CloseDate**, причём считаем уже по **самой свежей** на сегодня прогнозной кривой `@LastRep`.
   → усредняем `KEY_RATE` за диапазон
   `[@CloseDate ; @CloseDate + (@Term-1) дн.]`,
   но только в срезе `DT_REP = @LastRep`.

Такой диапазон содержит ровно `@Term` строк, поэтому обычное `AVG()` работает быстро и без дополнительной арифметики.

---

## Быстрая inline-табличная функция

Inline-TVF выполняется как часть оптимизированного плана запроса, не создаёт курсоров и, в отличие от скалярной функции, **не заставляет движок ходить построчно**.

```sql
CREATE OR ALTER FUNCTION ALM_TEST.WORK.ufn_GetDepositForecastRates
(
    @OpenDate  DATE,
    @Term      INT
)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
(
    /* 1. Самая свежая дата прогноза, доступная на момент вызова */
    WITH LastRep AS (
        SELECT MAX(DT_REP) AS LastRep
        FROM  ALM.info.VW_ForecastKEY_everyday WITH (NOLOCK)
        WHERE DT_REP <= CAST(GETDATE() AS DATE)
    )
    /* 2. Выносим расчётных помощников */
    , Params AS (
        SELECT 
            @OpenDate                       AS OpenDate ,
            DATEADD(day,@Term,@OpenDate)    AS CloseDate ,
            @Term                           AS Term
    )
    /* 3. Собираем итог одной строкой  */
    SELECT  
        /* --- средний прогноз на момент открытия --- */
        open_fk.AVG_KEY_RATE               AS AvgRateAtOpen ,
        /* --- средний прогноз на момент закрытия --- */
        ( SELECT AVG(fk.KEY_RATE)
          FROM  ALM.info.VW_ForecastKEY_everyday fk WITH (NOLOCK)
          CROSS JOIN LastRep lr
          CROSS JOIN Params  p
          WHERE fk.DT_REP = lr.LastRep
            AND fk.[Date] >= p.CloseDate
            AND fk.[Date] <  DATEADD(day, p.Term, p.CloseDate)  --  @Term дней
        )                                   AS AvgRateAtClose
    FROM  ALM.info.VW_ForecastKEY_everyday open_fk WITH (NOLOCK)
    CROSS JOIN Params p
    WHERE open_fk.DT_REP = p.OpenDate
      AND open_fk.Term   = p.Term
);
GO
```

### Как пользоваться

```sql
DECLARE @OpenDate DATE = '2025-04-20',
        @Term     INT  = 91;

SELECT *
FROM   ALM_TEST.WORK.ufn_GetDepositForecastRates(@OpenDate,@Term);
```

| AvgRateAtOpen | AvgRateAtClose |
| ------------- | -------------- |
| 0.1984…       | 0.1927…        |

*(цифры условные – возьмутся из актуального содержимого витрины)*

---

## Почему это работает быстро

1. **Фильтрация по индексируемым столбцам**
   У витрины обычно есть кластерный ключ `(DT_REP, [Date])`, поэтому оба подзапроса считывают только нужный диапазон без сканов.

2. **Нет курсоров и временных таблиц**
   Inline-TVF разворачивается в запрос-вызыватель; движок применяет push-down-предикаты и использует параллели, если нужно.

3. **Агрегация ровно по @Term строкам**
   Даже при годовом депозите это всего 365 строк, что мизерно по сравнению с объёмом витрины.

---

### Советы для встраивания в будущий процесс ETL

* **Снимайте снапшот** в рабочую таблицу сразу после расчёта:

  ```sql
  INSERT INTO ALM_TEST.WORK.DepositForecastSnapshot
        (OpenDate, Term, AvgRateAtOpen, AvgRateAtClose, SnapshotDttm)
  SELECT @OpenDate, @Term, AvgRateAtOpen, AvgRateAtClose, SYSUTCDATETIME()
  FROM   ALM_TEST.WORK.ufn_GetDepositForecastRates(@OpenDate,@Term);
  ```

* **Индексируйте** таблицу-приёмник по `(OpenDate, Term)` – так модель быстрее найдёт нужную строку.

* **Регулярно обновляйте @LastRep** (т.е. содержимое витрины), иначе `AvgRateAtClose` устареет; достаточно пере-инсерта после ежедневного пересчёта витрины.

---

> Если нужно вернуть больше поясняющих столбцов (например, дату закрытия или саму `@LastRep`), просто добавьте их в `SELECT` внутри функции – это не повлияет на производительность.
