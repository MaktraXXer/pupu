Ошибка «недопустимое имя столбца "rn"» появляется, потому что
в SQL Server нельзя обращаться к псевдониму столбца, объявленному
в этой же строке `SELECT`, – а мы делали это, вызывая
`DATEADD(day, rn, …)`.

Ниже – **исправленная, цельная версия процедуры** (все имена уникальны,
`rn` больше не используется внутри того же `SELECT`).

```sql
USE ALM_TEST;
GO
/*========================================================================
  Процедура: dbo.usp_ApplyKeyScenario
  Описание : Накладывает выбранный сценарий ставок на базовый кэш
             WORK.ForecastKey_Cache и сохраняет результат в
             WORK.ForecastKey_Cache_Scen.
========================================================================*/
IF OBJECT_ID('dbo.usp_ApplyKeyScenario','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_ApplyKeyScenario;
GO
CREATE PROCEDURE dbo.usp_ApplyKeyScenario
      @Scenario     tinyint      = 1,              -- 1 | 2 | 3 …
      @HistoryCut   date         = '2025-07-01',   -- с этой даты переписываем
      @NextCBDate   date         = '2026-08-17',   -- след. заседание (факт)
      @HorizonDays  int          = 200             -- сколько dt_rep вперёд
AS
SET NOCOUNT ON;

/*------------------------------------------------------------------*/
/* 0. опорные даты                                                  */
/*------------------------------------------------------------------*/
DECLARE
    @AnchorFact  date = (SELECT MAX(DT_REP)
                         FROM WORK.ForecastKey_Cache), -- t-1
    @LastSnap    date = DATEADD(day, @HorizonDays, @AnchorFact);

/*------------------------------------------------------------------*/
/* 1. таблица чисел  (2000-01-01 … 2030-12-31)                       */
/*------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#num','U') IS NOT NULL DROP TABLE #num;

;WITH seq AS (
     SELECT TOP (DATEDIFF(day,'2000-01-01','2031-01-01'))
            val = ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1
     FROM sys.all_objects a CROSS JOIN sys.all_objects b
)
SELECT n = val,
       d = DATEADD(day, val, '2000-01-01')
INTO   #num;
/*------------------------------------------------------------------*/
/* 2. разворачиваем сценарий «день-в-день» до (@NextCBDate-1)        */
/*------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#scen_day','U') IS NOT NULL DROP TABLE #scen_day;

;WITH bnd AS (
      SELECT change_dt,
             key_rate,
             nxt = LEAD(change_dt) OVER (ORDER BY change_dt)
      FROM   WORK.KeyRate_Scenarios
      WHERE  SCENARIO = @Scenario
)
SELECT DATEADD(day, n.n, b.change_dt) AS [Date],
       b.key_rate
INTO   #scen_day
FROM   bnd b
JOIN   #num n
  ON   n.d BETWEEN b.change_dt
              AND DATEADD(day, -1, ISNULL(b.nxt, DATEADD(day, -1, @NextCBDate)));

/*------------------------------------------------------------------*/
/* 3. базовый кэш для диапазона HistoryCut … LastSnap               */
/*------------------------------------------------------------------*/
;WITH base AS (
      SELECT *
      FROM   WORK.ForecastKey_Cache
      WHERE  DT_REP BETWEEN @HistoryCut AND @LastSnap
),
merged AS (   /* подменяем KEY_RATE по сценарию до @NextCBDate-1 */
      SELECT b.DT_REP,
             b.[Date],
             NewRate = COALESCE(s.key_rate, b.KEY_RATE)
      FROM   base b
      LEFT   JOIN #scen_day s ON s.[Date] = b.[Date]
      WHERE  b.[Date] < @NextCBDate

      UNION ALL               /* после @NextCBDate -> оставляем факт/прогноз */
      SELECT b.DT_REP, b.[Date], b.KEY_RATE
      FROM   base b
      WHERE  b.[Date] >= @NextCBDate
),
final AS (    /* пересчитываем TERM и AVG_KEY_RATE */
      SELECT m.DT_REP,
             m.[Date],
             m.NewRate                            AS KEY_RATE,
             TERM = ROW_NUMBER() OVER
                      (PARTITION BY m.DT_REP ORDER BY m.[Date]),
             AVG_KEY_RATE = AVG(m.NewRate) OVER
                      (PARTITION BY m.DT_REP
                       ORDER BY     m.[Date]
                       ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      FROM   merged m
)
/*------------------------------------------------------------------*/
/* 4. записываем результат                                           */
/*------------------------------------------------------------------*/
BEGIN TRAN;
    /* очищаем слой сценария */
    DELETE FROM WORK.ForecastKey_Cache_Scen
    WHERE  SCENARIO = @Scenario;

    /* добавляем историю до HistoryCut без изменений */
    INSERT INTO WORK.ForecastKey_Cache_Scen
           (SCENARIO, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE)
    SELECT @Scenario, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
    FROM   WORK.ForecastKey_Cache
    WHERE  DT_REP < @HistoryCut;

    /* добавляем пересчитанную часть */
    INSERT INTO WORK.ForecastKey_Cache_Scen
           (SCENARIO, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE)
    SELECT @Scenario, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
    FROM   final;
COMMIT TRAN;
GO
```

### Как запустить

```sql
/* 1. ваш «старый» скрипт создаёт WORK.ForecastKey_Cache */
-- ...

/* 2. применяем сценарий (пример для сценария 1) */
EXEC dbo.usp_ApplyKeyScenario @Scenario = 1;

/* при необходимости — сценарий 2 и 3 */
EXEC dbo.usp_ApplyKeyScenario @Scenario = 2;
EXEC dbo.usp_ApplyKeyScenario @Scenario = 3;
```

### Что проверять

```sql
/* всего 200 будущих DT_REP начиная с 2025-07-24 */
SELECT COUNT(DISTINCT DT_REP)
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND  DT_REP >= '2025-07-24';

/* ставка 18 % по сценарию 1 на 28-07-2025 */
SELECT *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND   DT_REP = '2025-07-24'   -- первый «будущий» срез
  AND   [Date] = '2025-07-28';

/* история до 30-06-2025 не тронута */
SELECT *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND   DT_REP = '2025-06-30';
```

Исправление в блоке создания #num убрало обращение к псевдониму
`rn` в той же строке, поэтому ошибка компиляции больше не возникает.
