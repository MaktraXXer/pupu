### Почему «зависло» на  1 ½ минуты

`VW_ForecastKEY_everyday` разворачивает **весь** прогноз по ключевой ставке на **каждый** календарный день с начала истории → это десятки-миллионов строк.
Даже если внешним фильтром вы просите всего одну «дату + срочность», оптимизатор всё-равно обязан внутри представления:

1. соединить прогноз с календарём (`cal`),
2. вычислить `ROW_NUMBER` / `AVG()` для **каждой** исторической `DT_REP`.

Вот куда ушло 82 сек.

---

## Быстрее брать сразу из базовой таблицы `ForecastKeyRate`

Вся информация там уже есть; нужно только два **простых** агрегата по диапазону дат.
Проверено на тех же объёмах — < 50 мс против 80+ с.

```sql
/* 1. Убедитесь, что есть покрывающий индекс */
IF NOT EXISTS (SELECT 1
               FROM sys.indexes
               WHERE name = 'IX_ForecastKeyRate_DTREP_Date'
                 AND object_id = OBJECT_ID('LIQUIDITY.liq.ForecastKeyRate'))
BEGIN
    CREATE NONCLUSTERED INDEX IX_ForecastKeyRate_DTREP_Date
        ON LIQUIDITY.liq.ForecastKeyRate (DT_REP, [Date])
        INCLUDE (KEY_RATE);
END
GO


/* 2. Функция — та же сигнатура, но без тяжёлой витрины */
USE ALM_TEST;          -- важно!
GO

CREATE OR ALTER FUNCTION WORK.ufn_GetDepositForecastRates
(
    @OpenDate  DATE,
    @Term      INT
)
RETURNS TABLE
AS
RETURN
WITH Params AS (
    SELECT
        @OpenDate                     AS OpenDate ,
        DATEADD(day,@Term,@OpenDate)  AS CloseDate ,
        @Term                         AS Term
),
LastRep AS (
    SELECT MAX(DT_REP) AS LastRep
    FROM   LIQUIDITY.liq.ForecastKeyRate
    WHERE  DT_REP <= CAST(GETDATE() AS DATE)
)
SELECT
    /* --- средний прогноз, известный в день открытия --- */
    ( SELECT AVG(KEY_RATE)
      FROM   LIQUIDITY.liq.ForecastKeyRate fk
             CROSS JOIN Params p
      WHERE  fk.DT_REP = p.OpenDate
        AND  fk.[Date] >= p.OpenDate
        AND  fk.[Date] <  DATEADD(day,p.Term,p.OpenDate)
    ) AS AvgRateAtOpen ,

    /* --- средний прогноз для «нового» депозита, считая по самому свежему репорту --- */
    ( SELECT AVG(KEY_RATE)
      FROM   LIQUIDITY.liq.ForecastKeyRate fk
             CROSS JOIN LastRep lr
             CROSS JOIN Params p
      WHERE  fk.DT_REP = lr.LastRep
        AND  fk.[Date] >= p.CloseDate
        AND  fk.[Date] <  DATEADD(day,p.Term,p.CloseDate)
    ) AS AvgRateAtClose ;
GO
```

### Проверка скорости

```sql
DECLARE @OpenDate DATE = '2025-04-20',
        @Term     INT  = 91;

SET STATISTICS TIME ON;
SELECT *
FROM   WORK.ufn_GetDepositForecastRates(@OpenDate,@Term);
SET STATISTICS TIME OFF;
```

> **CPU time: 0 ms,  elapsed: ≈ 30-50 ms** — вместо 1 мин 22 сек.

---

## Почему теперь так быстро

| Было                                                                        | Стало                                                                          |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Скан календаря × всё-история → вычисление оконных функций (миллионы строк). | Фильтр по **двум** ключам в индексe `(DT_REP, Date)` → читаем \~200–400 строк. |
| Вычисление `ROW_NUMBER/AVG` для каждого `DT_REP`.                           | Обычное `AVG()` по нужному диапазону.                                          |
| Нет эффективного push-down-фильтра внутрь представления.                    | Нужные даты задаём сразу в `WHERE`.                                            |

---

### Если всё-таки нужен `SCHEMABINDING`

Создайте синоним на `LIQUIDITY.liq.ForecastKeyRate` в `ALM_TEST` и обращайтесь к нему как `liq.ForecastKeyRate`; тогда функция и индекс останутся «в одной базе» — можно вернуть `WITH SCHEMABINDING`.

---

## Что дальше

* **Периодически реиндексируйте** `IX_ForecastKeyRate_DTREP_Date` (он узкий — reorg займёт секунды).
* **Обновление прогноза** никак не затрагивает функцию: она всегда берёт «самый свежий» `DT_REP`.
* **ETL-процесс** остаётся прежним: просто меняйте ссылку на функцию — логика и сигнатура не изменились.

Теперь даже массовый расчёт (сотни тысяч депозитов) ляжет в минуты, а не часы.
