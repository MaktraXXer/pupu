Вот минимальный, цельный скрипт-процедура, который **берёт уже готовый `WORK.ForecastKey_Cache`** (тот, что вы создаёте «старым» кодом),
накладывает на него *любой* из трёх сценариев, **пересчитывает TERM / AVG\_KEY\_RATE** и сохраняет результат в `WORK.ForecastKey_Cache_Scen`.

---

```sql
USE ALM_TEST;
GO
/*──────────────────────────────────────────────────────────────────────
  1. Входные справочники:
       • WORK.KeyRate_Scenarios      – сценарии (вы уже заполняете)
       • WORK.ForecastKey_Cache      – «базовый» кэш (ваш старый код)
  2. Выход:
       • WORK.ForecastKey_Cache_Scen – кэш с учётом сценария
──────────────────────────────────────────────────────────────────────*/

IF OBJECT_ID('dbo.usp_ApplyKeyScenario','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_ApplyKeyScenario;
GO
CREATE PROCEDURE dbo.usp_ApplyKeyScenario
      @Scenario      tinyint       = 1,             -- 1|2|3 …
      @HistoryCut    date          = '2025-07-01',  -- с этой датой переписываем
      @NextCBDate    date          = '2026-08-17',  -- след. заседание (факт)
      @HorizonDays   int           = 200            -- сколько dt_rep вперёд
AS
SET NOCOUNT ON;

/*--------------------------------------------------------------------
  0. сервисная таблица чисел (чтобы развернуть интервалы)
--------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#num') IS NOT NULL DROP TABLE #num;
;WITH seq AS (
     SELECT TOP (DATEDIFF(day,'2000-01-01','2031-01-01'))
            rn = ROW_NUMBER() OVER(ORDER BY (SELECT NULL))-1
     FROM sys.all_objects a CROSS JOIN sys.all_objects b
)
SELECT n = rn,
       d = DATEADD(day,rn,'2000-01-01')
INTO   #num;

/*--------------------------------------------------------------------
  1. разворачиваем выбранный сценарий «день-в-день»
--------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#scen_day') IS NOT NULL DROP TABLE #scen_day;

;WITH bnd AS (
      SELECT change_dt,
             key_rate,
             nxt = LEAD(change_dt) OVER (ORDER BY change_dt)
      FROM   WORK.KeyRate_Scenarios
      WHERE  SCENARIO = @Scenario
)
SELECT  DATEADD(day, n.n, b.change_dt)          AS [Date],
        b.key_rate                              AS key_rate
INTO    #scen_day
FROM   bnd b
JOIN   #num n
       ON n.d BETWEEN b.change_dt
                  AND DATEADD(day,-1,ISNULL(b.nxt,DATEADD(day,-1,@NextCBDate)));

/*--------------------------------------------------------------------
  2. ограничиваем базовый кэш «хвостом» в  @HorizonDays   дней
      (предполагается, что он уже создан вашим «старым» кодом)
--------------------------------------------------------------------*/
DECLARE @AnchorFact date =
        (SELECT MAX(DT_REP) FROM WORK.ForecastKey_Cache); -- t-1

DECLARE @LastSnap   date = DATEADD(day,@HorizonDays,@AnchorFact);

/*--------------------------------------------------------------------
  3. перестраиваем кэш:   старые строки + сценарий + пересчёт AVG
--------------------------------------------------------------------*/
DELETE FROM WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = @Scenario;                         -- чистим слой

;WITH base AS (      -- берём только нужный диапазон из «старого» кэша
      SELECT *
      FROM   WORK.ForecastKey_Cache
      WHERE  DT_REP BETWEEN @HistoryCut AND @LastSnap
),
merged AS (         -- подменяем KEY_RATE по сценарию
      SELECT
          b.DT_REP,
          b.[Date],
          NewRate = COALESCE(s.key_rate, b.KEY_RATE)  -- сценарий > факт
      FROM   base b
      LEFT   JOIN #scen_day s ON s.[Date] = b.[Date]
      WHERE  b.[Date] < @NextCBDate                  -- до заседания по сцен.
      UNION ALL                                      -- после – старый факт
      SELECT b.DT_REP, b.[Date], b.KEY_RATE
      FROM   base b
      WHERE  b.[Date] >= @NextCBDate
),
final AS (          -- пересчитываем TERM / AVG_KEY_RATE
      SELECT
          r.DT_REP,
          r.[Date],
          r.NewRate                                   AS KEY_RATE,
          TERM = ROW_NUMBER() OVER
                   (PARTITION BY r.DT_REP ORDER BY r.[Date]),
          AVG_KEY_RATE = AVG(r.NewRate) OVER
                   (PARTITION BY r.DT_REP
                    ORDER BY     r.[Date]
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      FROM   merged r
)
INSERT INTO WORK.ForecastKey_Cache_Scen
       (SCENARIO, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE)
SELECT @Scenario, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
FROM   final

/*--------------------------------------------------------------------
  4. добавляем нетронутую «доправочную» историю  (< @HistoryCut)
--------------------------------------------------------------------*/
INSERT INTO WORK.ForecastKey_Cache_Scen
       (SCENARIO, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE)
SELECT @Scenario, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
FROM   WORK.ForecastKey_Cache     -- оригинал
WHERE  DT_REP < @HistoryCut;
GO
```

### Как пользоваться

```sql
/* сначала ваш старый скрипт → WORK.ForecastKey_Cache */
-- …

/* затем применяем сценарии */
EXEC dbo.usp_ApplyKeyScenario @Scenario = 1;   -- «18/16/…»
EXEC dbo.usp_ApplyKeyScenario @Scenario = 2;   -- «17/15/…»
EXEC dbo.usp_ApplyKeyScenario @Scenario = 3;   -- «16/14/…»
```

### Проверки

```sql
/* 1. должно быть ровно 200 будущих DT_REP  (24.07.2025--08.02.2026) */
SELECT COUNT(DISTINCT DT_REP)
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND  DT_REP >= '2025-07-24';

/* 2. увидеть, что 28-07-25 у сценария 1 = 18 % */
SELECT TOP 10 *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND   DT_REP = '2025-07-24'      -- первый «будущий» срез
ORDER BY TERM;

/* 3. убедиться, что история до 30-06-25 не тронута */
SELECT *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND   DT_REP = '2025-06-30'
ORDER BY TERM;
```

### Что делает процедура

1. **Не трогает** всё, что моложе `@HistoryCut`.
2. Разворачивает ваш сценарий «день-в-день» до 16.08.2026.
3. Подменяет `KEY_RATE` в интервалах сценария; после 16.08.2026
   ставка остаётся «как в базовых данных».
4. Для каждого `DT_REP` (от `@HistoryCut` до t-1+`@HorizonDays`)
   заново считает `TERM` и `AVG_KEY_RATE`.
5. Сохраняет результат в `WORK.ForecastKey_Cache_Scen`
   с полем `SCENARIO`, чтобы разные сценарии жили параллельно.

> Вся «грязная» работа (замена ставок, перерасчёт средних, срезы на 200 дней вперёд) лежит в одной компактной процедуре, а ваш старый надёжный код продолжает формировать базовый кэш.
