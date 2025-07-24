Ниже ― полный рабочий текст **одной** процедуры `dbo.usp_BuildKeyCache`.
Я переписал его «с нуля», поэтому:

* **нет дублирования объявлений** переменных/таблиц;
* имена временных таблиц уникальны внутри процедуры;
* все переменные объявлены **до** первого использования;
* нет лишних `GO` внутри тела процедуры.

Проверьте, что объект-витрины `ALM.info.VW_ForecastKEY_interval` действительно содержит снимок **t-1** на `MAX(DT_REP)`.

```sql
USE ALM_TEST;
GO
IF OBJECT_ID('dbo.usp_BuildKeyCache','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_BuildKeyCache;
GO
/***********************************************************************
  dbo.usp_BuildKeyCache
     @Scenario      – 1,2,3 …
     @HistoryCut    – начиная с какой даты «переписываем» кэш
     @HorizonDays   – хвост после t-1
***********************************************************************/
CREATE PROCEDURE dbo.usp_BuildKeyCache
      @Scenario      tinyint,
      @HistoryCut    date = '2025-07-01',
      @HorizonDays   int  = 200
AS
SET NOCOUNT ON;

/*───────────────────────── 0. опорные даты ─────────────────────────*/
DECLARE
    @AnchorFact   date = (SELECT MAX(DT_REP)
                          FROM ALM.info.VW_ForecastKEY_interval),  -- t-1
    @ScenCutOff   date = '2026-08-16',  -- 16-08-2026 = день перед заседанием
    @LastSnapDate date = DATEADD(day,@HorizonDays,@AnchorFact);

/*────────────────── 1. таблица чисел 2000-01-01 … 2030-12-31 ─────────*/
IF OBJECT_ID('tempdb..#num') IS NOT NULL DROP TABLE #num;
;WITH seq AS (
     SELECT TOP (DATEDIFF(day,'2000-01-01','2030-12-31')+1)
            rn = ROW_NUMBER() OVER (ORDER BY (SELECT NULL))-1
     FROM sys.all_objects a CROSS JOIN sys.all_objects b
)
SELECT n   = rn,
       d   = DATEADD(day,rn,'2000-01-01')
INTO   #num
FROM   seq
OPTION (MAXRECURSION 0);      -- не жалуемся на TOP

/*────────────────── 2. перенесём «старую» историю как есть ───────────*/
DELETE FROM WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = @Scenario;

INSERT INTO WORK.ForecastKey_Cache_Scen
SELECT  @Scenario,
        DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
FROM    ALM.info.VW_ForecastKEY_everyday
WHERE   DT_REP < @HistoryCut;

/*────────────────── 3. сценарий: раскладываем «день-в-день» ─────────*/
IF OBJECT_ID('tempdb..#scen_day') IS NOT NULL DROP TABLE #scen_day;

;WITH bnd AS (
      SELECT change_dt,
             key_rate,
             nxt = LEAD(change_dt) OVER (ORDER BY change_dt)
      FROM   WORK.KeyRate_Scenarios
      WHERE  SCENARIO = @Scenario
),
days AS (
      SELECT DATEADD(day, n.n, b.change_dt) AS [Date],
             b.key_rate
      FROM   bnd b
      JOIN   #num n
             ON n.d BETWEEN b.change_dt
                        AND DATEADD(day,-1, ISNULL(b.nxt,@ScenCutOff))
)
SELECT  [Date], key_rate
INTO    #scen_day
FROM    days;

/*────────────────── 4. «факт на день» и прогноз t-1 ─────────────────*/
IF OBJECT_ID('tempdb..#tl') IS NOT NULL DROP TABLE #tl;

;WITH cal AS (
      SELECT d
      FROM   #num
      WHERE  d BETWEEN @HistoryCut AND @LastSnapDate
),
hist AS (                      -- факт: для каждой даты – ставка TERM=1
      SELECT DT_REP AS d, KEY_RATE
      FROM   ALM.info.VW_ForecastKEY_everyday
      WHERE  DT_REP BETWEEN @HistoryCut AND @AnchorFact
),
fore AS (                      -- прогноз t-1 на всё будущее
      SELECT [Date] AS d, KEY_RATE
      FROM   ALM.info.VW_ForecastKEY_interval
      WHERE  DT_REP = @AnchorFact
        AND  [Date] BETWEEN @AnchorFact AND @LastSnapDate
)
SELECT  c.d                        AS [Date],
        KEY_RATE = COALESCE(s.key_rate,    -- 1) сценарий
                            h.KEY_RATE,    -- 2) факт
                            f.KEY_RATE)    -- 3) прогноз t-1
INTO    #tl
FROM    cal c
LEFT    JOIN #scen_day s ON s.[Date] = c.d
LEFT    JOIN hist      h ON h.d      = c.d
LEFT    JOIN fore      f ON f.d      = c.d;

/*────────────────── 5. сформируем кэш для всех DT_REP ≥ HistoryCut ──*/
;WITH all_rep AS (
      SELECT d
      FROM   #num
      WHERE  d BETWEEN @HistoryCut AND @LastSnapDate
)
INSERT INTO WORK.ForecastKey_Cache_Scen
SELECT  @Scenario,
        r.d            AS DT_REP,
        t.[Date],
        t.KEY_RATE,
        TERM = ROW_NUMBER() OVER (PARTITION BY r.d ORDER BY t.[Date]),
        AVG_KEY_RATE = AVG(t.KEY_RATE) OVER (
                           PARTITION BY r.d
                           ORDER BY     t.[Date]
                           ROWS BETWEEN UNBOUNDED PRECEDING
                                AND   CURRENT ROW)
FROM   all_rep r
JOIN   #tl     t ON t.[Date] >= r.d;
GO
```

### Проверяем

```sql
EXEC dbo.usp_BuildKeyCache @Scenario = 1;

/* Дата до HistoryCut – история не тронута */
SELECT TOP 30 *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND  DT_REP   = '2025-07-01'
ORDER BY TERM;

/* Дата «накануне» 24-07-25 – ставка должна смениться 28-07-25 */
SELECT *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND  DT_REP   = '2025-07-24'
ORDER BY TERM;
```

* строки до 27-07-25 будут по 0.20
* c 28-07-25 – 0.18 (сценарий-1)
* `TERM` начинается с 1, `AVG_KEY_RATE` растёт нарастающим итогом.

Вызовите процедуру с `@Scenario = 2` и `3` – получите альтернативные кэши.
