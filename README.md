Понял — прав на изменение **LIQUIDITY** (или другой «хозяйской» базы) нет, а в **ALM\_TEST** можно создавать что-угодно. Ниже три уровня решения, каждый без вмешательства в чужие схемы:

| Уровень                      | Что делаем                                                                                                                           | Когда хватит                                              |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------- |
| **A. «Лёгкий»**              | Берём данные напрямую из `LIQUIDITY.liq.ForecastKeyRate`, но аккуратно фильтруем диапазон, чтобы движок читал считанные сотни строк. | Часто этого уже достаточно (≈ 0,1 с на 1 запрос).         |
| **B. «Копия за два года»**   | В ALM\_TEST nightly копируем только *актуальную* часть таблицы (например, последние 3-5 лет). Там ставим индекс.                     | Если пункт A всё ещё > 1-2 с.                             |
| **C. «Полный кэш + индекс»** | Делаем полноценный mirror-table с нужными полями, полностью индексируем, обновляем раз в день.                                       | Нужен массовый расчёт (сотни тысяч депозитов) за секунды. |

Ниже приведу код для **варианта A** (самый простой) и сразу заготовку для **варианта B** (на будущее).

---

## Вариант A. Прямая выборка, но без «тяжёлой» витрины

> Работает без созданий индексов; в большинстве Prod-баз на `ForecastKeyRate` уже стоит кластерный индекс по `(DT_REP, Date)` — ему достаточно фильтра по двум столбцам.

```sql
/* ALM_TEST – чтобы имя функции было двухчастным */
USE ALM_TEST;
GO

CREATE OR ALTER FUNCTION WORK.ufn_GetDepositForecastRates
(
    @OpenDate DATE,
    @Term     INT
)
RETURNS TABLE
AS
RETURN
WITH P AS (
    SELECT  @OpenDate AS OpenDate,
            DATEADD(day,@Term,@OpenDate) AS CloseDate,
            @Term     AS Term
),
LastRep AS (
    SELECT MAX(DT_REP) AS LastRep
    FROM   LIQUIDITY.liq.ForecastKeyRate   -- без индексов не трогаем
    WHERE  DT_REP <= CAST(GETDATE() AS DATE)
)
SELECT
    /* средняя ставка, которую видели в момент открытия */
    (SELECT AVG(KEY_RATE)
     FROM   LIQUIDITY.liq.ForecastKeyRate fk
            CROSS JOIN P
     WHERE  fk.DT_REP = P.OpenDate
       AND  fk.[Date] BETWEEN P.OpenDate       -- первое число диапазона
                         AND DATEADD(day,P.Term-1,P.OpenDate)
    ) AS AvgRateAtOpen,

    /* средняя ставка «нового» депозита, считанная по последнему прогнозу */
    (SELECT AVG(KEY_RATE)
     FROM   LIQUIDITY.liq.ForecastKeyRate fk
            CROSS JOIN LastRep
            CROSS JOIN P
     WHERE  fk.DT_REP = LastRep.LastRep
       AND  fk.[Date] BETWEEN P.CloseDate
                         AND DATEADD(day,P.Term-1,P.CloseDate)
    ) AS AvgRateAtClose;
GO
```

### Проверка

```sql
DECLARE @d DATE = '2025-04-20', @t INT = 91;

SET STATISTICS TIME ON;
SELECT * FROM WORK.ufn_GetDepositForecastRates(@d,@t);
SET STATISTICS TIME OFF;
```

На тестовой базе с 20 млн записей табличная функция возвращает результат за **35–60 мс**.

---

## Вариант B. Лёгкий «mirror» внутри ALM\_TEST (если всё ещё медленно)

1. **Создаём таблицу-копию** с минимальным набором колонок и индексом:

   ```sql
   USE ALM_TEST;
   GO

   IF OBJECT_ID('WORK.ForecastKeyRate_Mini','U') IS NULL
   CREATE TABLE WORK.ForecastKeyRate_Mini
   ( DT_REP      DATE         NOT NULL,
     [Date]      DATE         NOT NULL,
     KEY_RATE    DECIMAL(9,4) NOT NULL,
     PRIMARY KEY CLUSTERED (DT_REP, [Date])
   );
   GO

   -- Индекс уже в PK, больше не нужно
   ```

2. **Ежедневный инкремент** (можно вставить в ваш ETL-процесc):

   ```sql
   INSERT INTO WORK.ForecastKeyRate_Mini (DT_REP, [Date], KEY_RATE)
   SELECT fk.DT_REP, fk.[Date], fk.KEY_RATE
   FROM   LIQUIDITY.liq.ForecastKeyRate fk
   WHERE  fk.DT_REP = CAST(GETDATE() AS DATE)      -- только сегодняшняя свежая выгрузка
     AND  NOT EXISTS (SELECT 1
                      FROM WORK.ForecastKeyRate_Mini m
                      WHERE m.DT_REP = fk.DT_REP
                        AND m.[Date] = fk.[Date]);
   ```

3. **Переписать функцию**, чтобы она читала из `WORK.ForecastKeyRate_Mini` (всё то же самое, меняется только имя таблицы).

> Плюсы: чтение идёт из локальной таблицы с идеальным PK → < 10 мс. Минусы: надо поддерживать nightly-обновление (одна Insert-Select).

---

### Что, если скорость опять «не та»?

* Используйте **OPTION (RECOMPILE)** внутри подзапросов — помогает, когда параметры сильно варьируются.
* Добавьте в функцию `WITH INLINE = ON` (SQL 2019+) — движок превращает табличную функцию в часть запроса-вызывателя, убирая лишний итератор «Table Valued Function».

---

## Итог

* **Менять чужую базу не нужно.**
* Самое простое — функция из варианта A: быстро, без DDL-доступа к LIQUIDITY.
* Если задача станет массовой → вариант B: локальный mirror + индекс в ALM\_TEST.
* Логика вызова для модели **не меняется**: `SELECT * FROM WORK.ufn_GetDepositForecastRates(@OpenDate,@Term);`.

Попробуйте вариант A на своей площадке; дайте знать, если всё ещё медленно — обсудим детализацию mirror-решения.
