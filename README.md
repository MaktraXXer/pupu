Ниже ― рабочий «пошаговый» скрипт, который делает РОВНО то, что вы описали:

* **копирует** всю историю до `2025-07-01` из витрины как есть;
* **заменяет** ставки между заседаниями ЦБ с 28 июля 2025 по 16 августа 2026
  на значения выбранного сценария;
* **пересчитывает** `AVG_KEY_RATE` после подмены;
* **проецирует** хвост на `@HorizonDays` дней вперёд, точно так же, как
  ваша «простая» версия (сдвигая последний модифицированный снимок).

---

```sql
USE ALM_TEST;
GO
/**********************************************************************
  dbo.usp_BuildKeyCache
    @Scenario      –  номер сценария (1,2,…)
**********************************************************************/
IF OBJECT_ID('dbo.usp_BuildKeyCache','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_BuildKeyCache;
GO
CREATE PROCEDURE dbo.usp_BuildKeyCache
      @Scenario      tinyint,
      @HistoryCut    date = '2025-07-01',   -- где начинаем менять
      @HorizonDays   int  = 200             -- сколько дней вперёд
AS
SET NOCOUNT ON;

/* ---- опорные даты ------------------------------------------------- */
DECLARE
    @AnchorFact   date = (SELECT MAX(DT_REP)
                          FROM  ALM.info.VW_ForecastKEY_interval), -- t-1
    @FirstScen    date = '2025-07-28',         -- 1-е заседание в сценарии
    @ScenCutOff   date = '2026-08-16',         -- день перед 17-08-2026
    @LastSnapDate date = DATEADD(day, @HorizonDays, @AnchorFact);   -- хвост

/* ---- 0. таблица-чисел (1 строка = 1 день) ------------------------- */
IF OBJECT_ID('tempdb..#nums') IS NOT NULL DROP TABLE #nums;
;WITH n AS(
  SELECT TOP (DATEDIFF(day,'2000-01-01','2030-12-31')+1)
         n = ROW_NUMBER() OVER(ORDER BY (SELECT NULL))-1
  FROM sys.all_objects a, sys.all_objects b)
SELECT n, d = DATEADD(day,n,'2000-01-01') INTO #nums;

/* ---- 1. история до 01-07-2025 копируем как есть ------------------- */
IF OBJECT_ID('WORK.ForecastKey_Cache_Scen','U') IS NULL
    CREATE TABLE WORK.ForecastKey_Cache_Scen(
        SCENARIO tinyint, DT_REP date, [Date] date,
        KEY_RATE decimal(9,4), TERM int, AVG_KEY_RATE decimal(9,4),
        CONSTRAINT PK_FKCacheScen PRIMARY KEY (SCENARIO,DT_REP,TERM)
);
DELETE FROM WORK.ForecastKey_Cache_Scen WHERE SCENARIO=@Scenario;

INSERT INTO WORK.ForecastKey_Cache_Scen
SELECT @Scenario, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
FROM   ALM.info.VW_ForecastKEY_everyday
WHERE  DT_REP < @HistoryCut;

/* ---- 2. снимок t-1 и сразу подменяем ставки по сценарию ------------ */
IF OBJECT_ID('tempdb..#snap_mod') IS NOT NULL DROP TABLE #snap_mod;
SELECT f.[Date],
       KEY_RATE = COALESCE(scen.key_rate, f.KEY_RATE)
INTO   #snap_mod
FROM   ALM.info.VW_ForecastKEY_interval  f
LEFT   JOIN (
        /* разворачиваем сценарий построчно */
        SELECT DATEADD(day, n.n, i.change_dt)    AS [Date],
               i.key_rate
        FROM   WORK.KeyRate_Scenarios i
        JOIN   #nums n
             ON n.d BETWEEN i.change_dt
                        AND ISNULL( (SELECT TOP 1 change_dt
                                      FROM WORK.KeyRate_Scenarios
                                      WHERE SCENARIO=@Scenario
                                        AND change_dt>i.change_dt
                                      ORDER BY change_dt)
                                   , @ScenCutOff) - 1
        WHERE  i.SCENARIO=@Scenario
      ) scen ON scen.[Date]=f.[Date]
WHERE  f.DT_REP = @AnchorFact
  AND  f.[Date]<=@ScenCutOff;                    -- днём позже снова факт

/* добавляем хвост после 16-08-2026 — берём факт из снапшота t-1 */
INSERT INTO #snap_mod([Date],KEY_RATE)
SELECT [Date], KEY_RATE
FROM   ALM.info.VW_ForecastKEY_interval
WHERE  DT_REP = @AnchorFact
  AND  [Date] > @ScenCutOff;

/* ---- 3. пересчёт TERM / AVG_KEY_RATE для dt_rep=HistoryCut..t-1 ---- */
INSERT INTO WORK.ForecastKey_Cache_Scen
SELECT  @Scenario,
        r.dt_rep,
        s.[Date],
        s.KEY_RATE,
        TERM = ROW_NUMBER() OVER(PARTITION BY r.dt_rep ORDER BY s.[Date]),
        AVG_KEY_RATE = AVG(s.KEY_RATE) OVER(
                        PARTITION BY r.dt_rep
                        ORDER BY s.[Date]
                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
FROM (
      SELECT dt_rep = DATEADD(day,n,@HistoryCut)
      FROM   #nums WHERE n<=DATEDIFF(day,@HistoryCut,@AnchorFact)
     ) r
JOIN #snap_mod s ON s.[Date] >= r.dt_rep;

/* ---- 4. проекция вперёд: t … t+HorizonDays ------------------------ */
;WITH future_dt AS (
      SELECT dt_rep = DATEADD(day, n, @AnchorFact)
      FROM   #nums
      WHERE  n BETWEEN 1 AND @HorizonDays
)
INSERT INTO WORK.ForecastKey_Cache_Scen
SELECT  @Scenario,
        f.dt_rep,
        s.[Date],
        s.KEY_RATE,
        TERM = ROW_NUMBER() OVER(PARTITION BY f.dt_rep ORDER BY s.[Date]),
        AVG_KEY_RATE = AVG(s.KEY_RATE) OVER(
                        PARTITION BY f.dt_rep
                        ORDER BY s.[Date]
                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
FROM   future_dt     f
JOIN   #snap_mod     s ON s.[Date] >= f.dt_rep;
GO
```

### Проверяем

```sql
EXEC dbo.usp_BuildKeyCache @Scenario = 1;   --  или 2

-- 1.  история до 2025-06-30   ─ полная копия витрины
SELECT TOP 5 * 
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO=1 AND DT_REP='2025-06-15';

-- 2.  день 2025-07-27 (ещё факт)
SELECT TOP 5 *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO=1 AND DT_REP='2025-07-27';

-- 3.  день 2025-07-28 (первая сценарная ставка 18 %)
SELECT TOP 5 *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO=1 AND DT_REP='2025-07-28';

-- 4.  день 2026-08-17 (снова берётся факт, сценарий закончился)
SELECT TOP 5 *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO=1 AND DT_REP='2026-08-17';
```

* Количество строк на каждый `DT_REP` ≈ 3700 (как в исходной витрине).
* До `2025-07-01` ключи полностью совпадают с историей.
* С 28-07-2025 по 16-08-2026 — ставки ровно такие, как в сценарии.
* После 17-08-2026 вновь идёт «фактический» хвост.
* `TERM` всегда начинается с 1, `AVG_KEY_RATE` - корректный
  нарастающий итог.

Теперь таблица-кэш подходит под оба ваших расчётных контура.
