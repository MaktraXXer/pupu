Ошибка возникает из-за рекурсивного CTE в блоке разворачивания сценарной кривой. Вот исправленная версия процедуры с использованием итеративного подхода через таблицу чисел вместо рекурсии:

```sql
USE ALM_TEST
GO

/***********************************************************************
  dbo.usp_BuildKeyCache
    @Scenario      – номер сценария (1,2,…)
    @HistoryCut    – от какой даты хранить историю (по-умолч. '2025-07-01')
    @HorizonDays   – сколько дней после последнего факта держим хвост
***********************************************************************/
IF OBJECT_ID('dbo.usp_BuildKeyCache','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_BuildKeyCache;
GO
CREATE PROCEDURE dbo.usp_BuildKeyCache
      @Scenario      tinyint,
      @HistoryCut    date       = '2025-07-01',
      @HorizonDays   int        = 200
AS
SET NOCOUNT ON;

/* последний факт-прогноз */
DECLARE @AnchorFact  date = (SELECT MAX(DT_REP)
                             FROM  ALM.info.VW_ForecastKEY_interval);
DECLARE @HorizonEnd  date = DATEADD(day, @HorizonDays, @AnchorFact);

/* ------------------------------------------------------------------ */
/* 0. календарь с @HistoryCut по @HorizonEnd (без рекурсии)            */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
DECLARE @DaysCount int = DATEDIFF(day, @HistoryCut, @HorizonEnd);
SELECT TOP (@DaysCount + 1)
       d = DATEADD(day, ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1, @HistoryCut)
INTO   #cal
FROM   sys.all_objects a, sys.all_objects b;

/* ------------------------------------------------------------------ */
/* 1. базовый прогноз (витрина interval, снимок @AnchorFact)           */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('tempdb..#base_fore') IS NOT NULL DROP TABLE #base_fore;
SELECT f.[Date], f.KEY_RATE
INTO   #base_fore
FROM   ALM.info.VW_ForecastKEY_interval f
WHERE  f.DT_REP = @AnchorFact;

/* ------------------------------------------------------------------ */
/* 2. разворачиваем сценарную кривую (без рекурсии)                    */
/* ------------------------------------------------------------------ */
DECLARE @LastScenDate date = (
    SELECT MAX(change_dt)
    FROM   WORK.KeyRate_Scenarios
    WHERE  SCENARIO = @Scenario
);

IF OBJECT_ID('tempdb..#scen_rate') IS NOT NULL DROP TABLE #scen_rate;

;WITH s AS (
    SELECT change_dt,
           key_rate,
           nxt = LEAD(change_dt) OVER (ORDER BY change_dt)
    FROM   WORK.KeyRate_Scenarios
    WHERE  SCENARIO = @Scenario
),
Intervals AS (
    SELECT 
        change_dt AS start_date,
        ISNULL(DATEADD(day, -1, nxt), @HorizonEnd) AS end_date,
        key_rate
    FROM s
),
Numbers AS (
    SELECT TOP (DATEDIFF(day, @HistoryCut, @HorizonEnd) + 1) 
           n = ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1
    FROM   sys.all_objects a, sys.all_objects b
)
SELECT 
    DATEADD(day, n.n, i.start_date) AS [Date],
    i.key_rate
INTO #scen_rate
FROM Intervals i
CROSS APPLY (
    SELECT n
    FROM Numbers
    WHERE n <= DATEDIFF(day, i.start_date, i.end_date)
) n

UNION ALL

SELECT c.d, k.key_rate
FROM   #cal c
CROSS JOIN (SELECT key_rate
            FROM   WORK.KeyRate_Scenarios
            WHERE  SCENARIO = @Scenario
            AND    change_dt = @LastScenDate) k
WHERE  c.d > @LastScenDate
OPTION (MAXRECURSION 0);

/* ------------------------------------------------------------------ */
/* 3. итоговая временная шкала                                         */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('tempdb..#timeline') IS NOT NULL DROP TABLE #timeline;
SELECT  c.d AS [Date],
        KEY_RATE = CASE
            WHEN c.d <= @AnchorFact THEN COALESCE(h.KEY_RATE, b.KEY_RATE)
            ELSE COALESCE(s.key_rate, b.KEY_RATE)
        END
INTO    #timeline
FROM    #cal c
LEFT JOIN ALM.info.VW_ForecastKEY_everyday h
       ON h.DT_REP = c.d
LEFT JOIN #scen_rate s
       ON s.[Date] = c.d
LEFT JOIN #base_fore b
       ON b.[Date] = c.d;

/* ------------------------------------------------------------------ */
/* 4. список DT_REP, для которых делаем снимок                         */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('tempdb..#dt_rep') IS NOT NULL DROP TABLE #dt_rep;
SELECT DT_REP = d
INTO   #dt_rep
FROM   #cal
WHERE  d >= @HistoryCut;

/* ------------------------------------------------------------------ */
/* 5. сохраняем в кэш (с правильным расчетом среднего)                 */
/* ------------------------------------------------------------------ */
DELETE FROM WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = @Scenario;

;WITH CacheData AS (
    SELECT 
        @Scenario AS SCENARIO,
        r.DT_REP,
        t.[Date],
        t.KEY_RATE,
        TERM = ROW_NUMBER() OVER (PARTITION BY r.DT_REP ORDER BY t.[Date]),
        AVG_KEY_RATE = AVG(t.KEY_RATE) OVER (
            PARTITION BY r.DT_REP 
            ORDER BY t.[Date] 
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        )
    FROM #dt_rep r
    JOIN #timeline t ON t.[Date] >= r.DT_REP
)
INSERT INTO WORK.ForecastKey_Cache_Scen
SELECT SCENARIO, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
FROM CacheData
WHERE DT_REP >= @HistoryCut;
GO
```

### Основные изменения:

1. **Замена рекурсивных CTE на итеративные подходы**:
   - Генерация календаря через `ROW_NUMBER()` и `sys.all_objects`
   - Разворачивание интервалов через таблицу чисел вместо рекурсии

2. **Упрощенная логика разворачивания сценарной кривой**:
   - Сначала определяем интервалы между заседаниями
   - Затем генерируем все даты в каждом интервале через таблицу чисел
   - Добавляем хвост после последнего заседания

3. **Устранение ограничения MAXRECURSION**:
   - Все генерации дат работают через табличные методы
   - Убраны рекурсивные CTE, вызывавшие ошибку

4. **Сохраняем все основные функции**:
   - Корректное построение временной шкалы
   - Правильный расчет скользящего среднего
   - Поддержка исторических и сценарных данных

### Как использовать:

```sql
-- Очистка предыдущих данных (если нужно)
TRUNCATE TABLE WORK.ForecastKey_Cache_Scen;

-- Запуск для сценария 1
EXEC dbo.usp_BuildKeyCache @Scenario = 1;

-- Проверка результата
SELECT TOP 100 * 
FROM WORK.ForecastKey_Cache_Scen
WHERE SCENARIO = 1
ORDER BY DT_REP, TERM;
```

Этот код должен работать без ошибок и генерировать корректные данные для ваших методик прогнозирования.
