USE ALM_TEST;
GO
/*──────────────────────────────────────────────────────────────────────
  1. Входные справочники:
       • WORK.KeyRate_Scenarios      – сценарии
       • WORK.ForecastKey_Cache      – «базовый» кэш
  2. Выход:
       • WORK.ForecastKey_Cache_Scen – кэш с учётом сценария
──────────────────────────────────────────────────────────────────────*/

IF OBJECT_ID('dbo.usp_ApplyKeyScenario','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_ApplyKeyScenario;
GO
CREATE PROCEDURE dbo.usp_ApplyKeyScenario
      @Scenario      tinyint = 1,            -- 1|2|3 …
      @HistoryCut    date    = '2025-07-01',-- с этой датой переписываем
      @NextCBDate    date    = '2026-08-17',-- след. заседание (факт)
      @HorizonDays   int     = 200          -- сколько dt_rep вперёд
AS
BEGIN
    SET NOCOUNT ON;

    /*----------------------------------------------------------------
      0. сервисная таблица чисел (чтобы развернуть интервалы)
    ----------------------------------------------------------------*/
    IF OBJECT_ID('tempdb..#num') IS NOT NULL DROP TABLE #num;

    ;WITH seq AS (
        SELECT TOP (DATEDIFF(DAY,'2000-01-01','2031-01-01'))
               rn = ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1
        FROM sys.all_objects      a
        CROSS JOIN sys.all_objects b
    )
    SELECT
        n = rn,
        d = DATEADD(DAY, rn, '2000-01-01')
    INTO #num
    FROM seq;                                -- <-- пропущенный FROM

    /*----------------------------------------------------------------
      1. разворачиваем выбранный сценарий «день-в-день»
    ----------------------------------------------------------------*/
    IF OBJECT_ID('tempdb..#scen_day') IS NOT NULL DROP TABLE #scen_day;

    ;WITH bnd AS (
        SELECT
            change_dt,
            key_rate,
            nxt = LEAD(change_dt) OVER (ORDER BY change_dt)
        FROM WORK.KeyRate_Scenarios
        WHERE SCENARIO = @Scenario
    )
    SELECT
        [Date]   = DATEADD(DAY, n.n, b.change_dt),
        key_rate = b.key_rate
    INTO #scen_day
    FROM bnd b
    JOIN #num n
      ON n.d BETWEEN b.change_dt
                     AND DATEADD(DAY, -1,
                                 ISNULL(b.nxt,
                                        DATEADD(DAY, -1, @NextCBDate)));

    /*----------------------------------------------------------------
      2. ограничиваем базовый кэш «хвостом» в  @HorizonDays   дней
    ----------------------------------------------------------------*/
    DECLARE @AnchorFact date =
        (SELECT MAX(DT_REP) FROM WORK.ForecastKey_Cache);   -- t-1

    DECLARE @LastSnap  date = DATEADD(DAY, @HorizonDays, @AnchorFact);

    /*----------------------------------------------------------------
      3. перестраиваем кэш: старые строки + сценарий + пересчёт AVG
    ----------------------------------------------------------------*/
    DELETE FROM WORK.ForecastKey_Cache_Scen
    WHERE  SCENARIO = @Scenario;             -- очищаем слой-приёмник

    ;WITH base AS (          -- только нужный диапазон из «старого» кэша
        SELECT *
        FROM WORK.ForecastKey_Cache
        WHERE DT_REP BETWEEN @HistoryCut AND @LastSnap
    ),
    merged AS (             -- подменяем KEY_RATE по сценарию
        SELECT
            b.DT_REP,
            b.[Date],
            NewRate = COALESCE(s.key_rate, b.KEY_RATE) -- сценарий > факт
        FROM base b
        LEFT JOIN #scen_day s
               ON s.[Date] = b.[Date]
        WHERE b.[Date] < @NextCBDate                    -- до заседания
        UNION ALL                                       -- после – факт
        SELECT
            b.DT_REP,
            b.[Date],
            b.KEY_RATE
        FROM base b
        WHERE b.[Date] >= @NextCBDate
    ),
    final AS (              -- пересчитываем TERM / AVG_KEY_RATE
        SELECT
            r.DT_REP,
            r.[Date],
            KEY_RATE     = r.NewRate,
            TERM         = ROW_NUMBER() OVER
                             (PARTITION BY r.DT_REP ORDER BY r.[Date]),
            AVG_KEY_RATE = AVG(r.NewRate) OVER
                             (PARTITION BY r.DT_REP
                              ORDER BY r.[Date]
                              ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        FROM merged r
    )
    INSERT INTO WORK.ForecastKey_Cache_Scen
           (SCENARIO, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE)
    SELECT  @Scenario,
            DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
    FROM final;

    /*----------------------------------------------------------------
      4. добавляем нетронутую «доправочную» историю (< @HistoryCut)
    ----------------------------------------------------------------*/
    INSERT INTO WORK.ForecastKey_Cache_Scen
           (SCENARIO, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE)
    SELECT  @Scenario,
            DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
    FROM WORK.ForecastKey_Cache
    WHERE DT_REP < @HistoryCut;
END
GO


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
