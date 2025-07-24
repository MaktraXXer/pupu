### Почему данные «обрезались»

1. **#snap\_mod бралась только из витрины interval на DT\_REP = @AnchorFact**
   → в ней нет дат до 23-07-2025, поэтому в кэше для `DT_REP=2025-07-01`
   вы увидели лишь хвост, начинающийся 23 июля.

2. **Сценарные ставки подменялись только в этом одном снапшоте**,
   а для всех `DT_REP < @AnchorFact` в кэш попал тот же «обрезанный» хвост
   без применения сценариев.

Ниже процедура, которая:

* **копирует** историю *как есть* до `@HistoryCut`
  (`2025-07-01` по-умолчанию);
* строит **единый тайм-лайн** `KEY_RATE` на диапазон
  `@HistoryCut … @LastSnapDate`

  ```
  приоритет:  сценарий
              ↓
              история (VW_ForecastKEY_everyday  – один день = один снимок)
              ↓
              прогноз t-1  (VW_ForecastKEY_interval, DT_REP = @AnchorFact)
  ```
* заново считает `TERM` и `AVG_KEY_RATE` для каждого `DT_REP`
  от `@HistoryCut` до `@AnchorFact+@HorizonDays`.

---

```sql
USE ALM_TEST;
GO
IF OBJECT_ID('dbo.usp_BuildKeyCache','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_BuildKeyCache;
GO
CREATE PROCEDURE dbo.usp_BuildKeyCache
      @Scenario      tinyint,                 -- 1 / 2 / 3 …
      @HistoryCut    date = '2025-07-01',     -- откуда начинаем править
      @HorizonDays   int  = 200               -- сколько дней вперёд
AS
SET NOCOUNT ON;

----------------------------------------------------------------------
-- 0. опорные даты и «таблица чисел»
----------------------------------------------------------------------
DECLARE
    @AnchorFact   date = (SELECT MAX(DT_REP)
                          FROM   ALM.info.VW_ForecastKEY_interval), -- t–1
    @ScenCutOff   date = '2026-08-16',      -- день перед след. заседанием
    @LastSnapDate date = DATEADD(day,@HorizonDays,@AnchorFact);

IF OBJECT_ID('tempdb..#nums') IS NOT NULL DROP TABLE #nums;
;WITH n AS (
    SELECT TOP (DATEDIFF(day,'2000-01-01','2030-12-31')+1)
           n = ROW_NUMBER() OVER(ORDER BY (SELECT NULL))-1
    FROM sys.all_objects a CROSS JOIN sys.all_objects b
)
SELECT n,
       d = DATEADD(day,n,'2000-01-01')
INTO   #nums;

----------------------------------------------------------------------
-- 1. чистим сценарный кэш и переносим всю «доправочную» историю
----------------------------------------------------------------------
DELETE FROM WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = @Scenario;

INSERT INTO WORK.ForecastKey_Cache_Scen
SELECT  @Scenario,
        DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
FROM    ALM.info.VW_ForecastKEY_everyday
WHERE   DT_REP < @HistoryCut;               -- без изменений!

----------------------------------------------------------------------
-- 2. строим сценарную линию «день-в-день» на FirstScen … CutOff
----------------------------------------------------------------------
IF OBJECT_ID('tempdb..#scen_day') IS NOT NULL DROP TABLE #scen_day;

;WITH bnd AS (                               -- интервалы сценария
      SELECT k.change_dt,
             k.key_rate,
             nxt = LEAD(k.change_dt)
                   OVER (PARTITION BY k.SCENARIO ORDER BY k.change_dt)
      FROM   WORK.KeyRate_Scenarios k
      WHERE  k.SCENARIO = @Scenario
),
days AS (
      SELECT DATEADD(day, n.n, b.change_dt) AS [Date],
             b.key_rate
      FROM   bnd b
      JOIN   #nums n
             ON n.d BETWEEN b.change_dt
                        AND DATEADD(day,-1,ISNULL(b.nxt,@ScenCutOff))
)
SELECT [Date], key_rate
INTO   #scen_day
FROM   days;

----------------------------------------------------------------------
-- 3. единый тайм-лайн на диапазон   @HistoryCut … @LastSnapDate
----------------------------------------------------------------------
IF OBJECT_ID('tempdb..#timeline') IS NOT NULL DROP TABLE #timeline;

;WITH cal AS (
      SELECT d
      FROM   #nums
      WHERE  d BETWEEN @HistoryCut AND @LastSnapDate
),
hist_one_day AS (                          -- «факт» на свой же день
      SELECT DT_REP, KEY_RATE
      FROM   ALM.info.VW_ForecastKEY_everyday
      WHERE  DT_REP BETWEEN @HistoryCut AND @AnchorFact
),
fore_t1 AS (                               -- прогноз t-1 на все даты
      SELECT [Date], KEY_RATE
      FROM   ALM.info.VW_ForecastKEY_interval
      WHERE  DT_REP = @AnchorFact
        AND [Date] BETWEEN @AnchorFact AND @LastSnapDate
)
SELECT  c.d AS [Date],
        KEY_RATE = COALESCE(s.key_rate,
                            h.KEY_RATE,
                            f.KEY_RATE)
INTO    #timeline
FROM    cal c
LEFT    JOIN #scen_day  s ON s.[Date] = c.d            -- ← сценарий
LEFT    JOIN hist_one_day h ON h.DT_REP = c.d          -- ← история
LEFT    JOIN fore_t1     f ON f.[Date] = c.d;          -- ← прогноз t-1

----------------------------------------------------------------------
-- 4. формируем кэш для dt_rep = @HistoryCut … @LastSnapDate
----------------------------------------------------------------------
;WITH all_dt_rep AS (
      SELECT d
      FROM   #nums
      WHERE  d BETWEEN @HistoryCut AND @LastSnapDate
)
INSERT INTO WORK.ForecastKey_Cache_Scen
SELECT  @Scenario,
        r.d                 AS DT_REP,
        t.[Date],
        t.KEY_RATE,
        TERM = ROW_NUMBER() OVER (PARTITION BY r.d ORDER BY t.[Date]),
        AVG_KEY_RATE = AVG(t.KEY_RATE) OVER (
                        PARTITION BY r.d
                        ORDER BY     t.[Date]
                        ROWS BETWEEN UNBOUNDED PRECEDING
                             AND   CURRENT ROW)
FROM   all_dt_rep r
JOIN   #timeline  t
       ON t.[Date] >= r.d;                -- берём хвост от DT_REP и далее
GO
```

### Проверка

```sql
EXEC dbo.usp_BuildKeyCache @Scenario = 1;   -- / 2 / 3

-- исторический день: должен совпасть с витриной целиком
SELECT TOP 35 *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND  DT_REP   = '2025-07-01'
ORDER BY TERM;

-- «переходный» день:  ориентир— 28-07-2025 ставка 18 %
SELECT *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND  DT_REP   = '2025-07-24'
ORDER BY TERM;
```

* для `DT_REP = 2025-07-01` — строки идут с `2025-07-01`, ставка 0.20 до 27-07-25, с 28-07-25 - 0.18, `TERM` = 1, 2, …
* для `DT_REP = 2025-07-24` — 24–27 июля  0.20, 28 июля  0.18, `AVG_KEY_RATE` считается нарастающим.

Сценарии 2 и 3 создаются тем же вызовом процедуры с другим `@Scenario`.
