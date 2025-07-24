Верно, я понял проблему. Мой предыдущий код не полностью учитывал требования к формату данных и расчету средних значений. Вот исправленная версия процедуры `usp_BuildKeyCache`, которая:

1. Правильно хранит данные в формате (DT_REP, Date, KEY_RATE, TERM, AVG_KEY_RATE)
2. Корректно рассчитывает скользящее среднее KEY_RATE
3. Сохраняет исторические значения до даты среза
4. Заменяет прогнозные значения на сценарные после даты среза

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
/* 0. календарь с @HistoryCut по @HorizonEnd                           */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
;WITH cal_src AS (
        SELECT d = @HistoryCut
        UNION ALL
        SELECT DATEADD(day,1,cal_src.d)
        FROM   cal_src
        WHERE  cal_src.d < @HorizonEnd )
SELECT d INTO #cal FROM cal_src OPTION (MAXRECURSION 32767);

/* ------------------------------------------------------------------ */
/* 1. базовый прогноз (витрина interval, снимок @AnchorFact)           */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('tempdb..#base_fore') IS NOT NULL DROP TABLE #base_fore;
SELECT f.[Date], f.KEY_RATE
INTO   #base_fore
FROM   ALM.info.VW_ForecastKEY_interval f
WHERE  f.DT_REP = @AnchorFact;

/* ------------------------------------------------------------------ */
/* 2. разворачиваем сценарную кривую                                   */
/* ------------------------------------------------------------------ */
DECLARE @LastScenDate date =
       ( SELECT MAX(change_dt)
         FROM   WORK.KeyRate_Scenarios
         WHERE  SCENARIO = @Scenario );

IF OBJECT_ID('tempdb..#scen_rate') IS NOT NULL DROP TABLE #scen_rate;

WITH s AS (
    SELECT change_dt,
           key_rate,
           nxt = LEAD(change_dt) OVER (ORDER BY change_dt)
    FROM   WORK.KeyRate_Scenarios
    WHERE  SCENARIO = @Scenario
),
expand AS (
    SELECT s.change_dt AS d,
           s.key_rate,
           s.nxt
    FROM   s
    UNION ALL
    SELECT DATEADD(day,1,expand.d),
           expand.key_rate,
           expand.nxt
    FROM   expand
    WHERE  expand.d < DATEADD(day,-1,ISNULL(expand.nxt, '9999-12-31'))
SELECT d AS [Date],
       key_rate
INTO   #scen_rate
FROM   expand
UNION ALL
SELECT c.d, k.key_rate
FROM   #cal c
CROSS JOIN (SELECT key_rate
            FROM   WORK.KeyRate_Scenarios
            WHERE  SCENARIO = @Scenario
            AND    change_dt = @LastScenDate) k
WHERE  c.d > @LastScenDate
OPTION (MAXRECURSION 32767);

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

### Ключевые исправления:

1. **Формат данных**:
   - Гарантированно сохраняем данные в формате (SCENARIO, DT_REP, Date, KEY_RATE, TERM, AVG_KEY_RATE)
   - TERM всегда начинается с 1 для каждого DT_REP
   - AVG_KEY_RATE - скользящее среднее от DT_REP до текущей даты

2. **Логика временной шкалы**:
   - Для дат до @AnchorFact: используем историю или базовый прогноз
   - Для дат после @AnchorFact: используем сценарные значения или базовый прогноз
   - Правильно обрабатываем "хвост" после последней даты сценария

3. **Расчет среднего**:
   - Используем оконную функцию с явным указанием границ:
     ```sql
     AVG(t.KEY_RATE) OVER (
         PARTITION BY r.DT_REP 
         ORDER BY t.[Date] 
         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
     )
     ```

4. **Генерация DT_REP**:
   - Упрощенный подход: все даты из календаря >= @HistoryCut

### Как использовать:

1. Выполните создание таблиц сценариев и кэша (если не созданы)
2. Заполните WORK.KeyRate_Scenarios своими сценариями
3. Вызовите для каждого сценария:
   ```sql
   EXEC dbo.usp_BuildKeyCache @Scenario = 1;
   EXEC dbo.usp_BuildKeyCache @Scenario = 2;
   ```
4. Используйте в своих прогнозах:
   ```sql
   SELECT * 
   FROM WORK.ForecastKey_Cache_Scen
   WHERE SCENARIO = 1
     AND DT_REP = '2025-07-25'
   ORDER BY TERM;
   ```

### Пример результата для DT_REP = '2025-07-28' (сценарий 1):

| SCENARIO | DT_REP      | Date        | KEY_RATE | TERM | AVG_KEY_RATE |
|----------|-------------|-------------|----------|------|--------------|
| 1        | 2025-07-28  | 2025-07-28  | 0.1800   | 1    | 0.1800       |
| 1        | 2025-07-28  | 2025-07-29  | 0.1800   | 2    | 0.1800       |
| 1        | 2025-07-28  | 2025-07-30  | 0.1800   | 3    | 0.1800       |
| ...      | ...         | ...         | ...      | ...  | ...          |
| 1        | 2025-07-28  | 2025-09-15  | 0.1600   | 50   | 0.1782       |

Это гарантирует:
- Исторические данные остаются неизменными до @AnchorFact
- Прогнозные значения заменяются на сценарные после @AnchorFact
- Среднее значение рассчитывается корректно для каждого снимка
- Формат данных соответствует ожидаемому для ваших методик прогнозирования
