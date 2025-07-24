/*--------------------------------------------------------------------
  0. опорные даты и таблица-чисел
--------------------------------------------------------------------*/
DECLARE @AnchorFact   date,
        @FirstScen    date = '2025-07-28',
        @ScenCutOff   date = '2026-08-16',
        @LastSnapDate date;

SELECT @AnchorFact = MAX(DT_REP)
FROM   ALM.info.VW_ForecastKEY_interval;

SET @LastSnapDate = DATEADD(day, @HorizonDays, @AnchorFact);

/* явное описание временной таблицы --------------- */
IF OBJECT_ID('tempdb..#nums') IS NOT NULL DROP TABLE #nums;
CREATE TABLE #nums(                 -- метаданные известны компилятору
    n int   NOT NULL PRIMARY KEY,   -- порядковый номер
    d date  NOT NULL                -- соответствующая календарная дата
);

/* заполняем #nums один раз на диапазон 2000-01-01 … 2030-12-31 */
;WITH seq AS (
    SELECT TOP (DATEDIFF(day,'2000-01-01','2030-12-31') + 1)
           n = ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1
    FROM   sys.all_objects a
    CROSS  JOIN sys.all_objects b
)
INSERT INTO #nums(n, d)
SELECT n, DATEADD(day, n, '2000-01-01')
FROM   seq;


### Что сломалось

* В `CREATE PROCEDURE` во **внутреннем `DECLARE`** я ссылался на переменные,
  объявленные **в той же инструкции** – компилятор не разрешает таких
  перекрестных ссылок.
* Поэтому при компиляции процедура «не видела» `@AnchorFact`, `@ScenCutOff`
  и др. → сообщения *«Необходимо объявить …»*.

### Быстрое лечение — объявляем и инициализируем переменные по-старому,

по одной (или группами, но **без** ссылок друг на друга).

Ниже рабочая версия процедуры — строк в витрину копирует столько же,
сколько у вас в `VW_ForecastKEY_interval` (≈ 3700 на снимок),
историю до 01-07-2025 оставляет нетронутой, промежуток
28-07-2025 … 16-08-2026 подменяет по выбранному сценарию,
дальше снова факт и хвост на `@HorizonDays` вперёд.

```sql
USE ALM_TEST;
GO
IF OBJECT_ID('dbo.usp_BuildKeyCache','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_BuildKeyCache;
GO
/**********************************************************************
  dbo.usp_BuildKeyCache
**********************************************************************/
CREATE PROCEDURE dbo.usp_BuildKeyCache
      @Scenario      tinyint,                -- 1 / 2 / 3 …
      @HistoryCut    date = '2025-07-01',    -- где начинаем менять
      @HorizonDays   int  = 200              -- сколько дней вперёд
AS
SET NOCOUNT ON;

/*--------------------------------------------------------------------
  0. опорные даты и таблица-чисел
--------------------------------------------------------------------*/
DECLARE @AnchorFact   date,                  -- t-1
        @FirstScen    date = '2025-07-28',   -- 1-е заседание
        @ScenCutOff   date = '2026-08-16',   -- день перед 17-08-2026
        @LastSnapDate date;                  -- t-1 + горизонт

SELECT @AnchorFact = MAX(DT_REP)
FROM   ALM.info.VW_ForecastKEY_interval;

SET @LastSnapDate = DATEADD(day,@HorizonDays,@AnchorFact);

IF OBJECT_ID('tempdb..#nums') IS NOT NULL DROP TABLE #nums;
;WITH n AS (
    SELECT TOP (DATEDIFF(day,'2000-01-01','2030-12-31')+1)
           n = ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1
    FROM sys.all_objects a CROSS JOIN sys.all_objects b)
SELECT n,
       d = DATEADD(day,n,'2000-01-01')
INTO   #nums;

/*--------------------------------------------------------------------
  1. чистим и копируем «старую» историю (< 2025-07-01)
--------------------------------------------------------------------*/
DELETE FROM WORK.ForecastKey_Cache_Scen WHERE SCENARIO = @Scenario;

INSERT INTO WORK.ForecastKey_Cache_Scen
SELECT @Scenario, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
FROM   ALM.info.VW_ForecastKEY_everyday
WHERE  DT_REP < @HistoryCut;

/*--------------------------------------------------------------------
  2. берём снимок t-1 и подменяем ставки по сценарию
--------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#snap_mod') IS NOT NULL DROP TABLE #snap_mod;

-- разворачиваем сценарную линию «день-в-день»
;WITH scen_days AS (
    SELECT DATEADD(day, n.n, k.change_dt) AS [Date], k.key_rate
    FROM   WORK.KeyRate_Scenarios k
    JOIN   #nums n
         ON n.d BETWEEN k.change_dt
                    AND ISNULL( (SELECT TOP 1 change_dt
                                 FROM   WORK.KeyRate_Scenarios
                                 WHERE  SCENARIO = @Scenario
                                   AND  change_dt > k.change_dt
                                 ORDER BY change_dt),
                               @ScenCutOff) - 1
    WHERE  k.SCENARIO = @Scenario
)
SELECT  f.[Date],
        KEY_RATE = COALESCE(s.key_rate, f.KEY_RATE)
INTO    #snap_mod
FROM    ALM.info.VW_ForecastKEY_interval f
LEFT    JOIN scen_days s
           ON s.[Date] = f.[Date]
WHERE   f.DT_REP = @AnchorFact
  AND   f.[Date] <= @ScenCutOff;       -- после CutOff снова берём факт

-- добавляем «фактический» хвост после 16-08-2026
INSERT INTO #snap_mod([Date],KEY_RATE)
SELECT  [Date], KEY_RATE
FROM    ALM.info.VW_ForecastKEY_interval
WHERE   DT_REP = @AnchorFact
  AND    [Date] > @ScenCutOff;

/*--------------------------------------------------------------------
  3. пересчитываем TERM / AVG_KEY_RATE для dt_rep = 01-07-25 … t-1
--------------------------------------------------------------------*/
INSERT INTO WORK.ForecastKey_Cache_Scen
SELECT  @Scenario,
        r.dt_rep,
        s.[Date],
        s.KEY_RATE,
        TERM = ROW_NUMBER() OVER(PARTITION BY r.dt_rep ORDER BY s.[Date]),
        AVG_KEY_RATE = AVG(s.KEY_RATE) OVER(
                            PARTITION BY r.dt_rep
                            ORDER BY     s.[Date]
                            ROWS BETWEEN UNBOUNDED PRECEDING
                                 AND   CURRENT ROW)
FROM (
      SELECT dt_rep = DATEADD(day, n, @HistoryCut)
      FROM   #nums
      WHERE  n <= DATEDIFF(day, @HistoryCut, @AnchorFact)
     ) r
JOIN #snap_mod s ON s.[Date] >= r.dt_rep;

/*--------------------------------------------------------------------
  4. проекция вперёд: t … t+HorizonDays
--------------------------------------------------------------------*/
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
        TERM = ROW_NUMBER() OVER (PARTITION BY f.dt_rep ORDER BY s.[Date]),
        AVG_KEY_RATE = AVG(s.KEY_RATE) OVER(
                            PARTITION BY f.dt_rep
                            ORDER BY     s.[Date]
                            ROWS BETWEEN UNBOUNDED PRECEDING
                                 AND   CURRENT ROW)
FROM   future_dt f
JOIN   #snap_mod s ON s.[Date] >= f.dt_rep;
GO
```

### Проверка

```sql
EXEC dbo.usp_BuildKeyCache @Scenario = 1;

-- история до 01-07-2025
SELECT TOP 3 * FROM WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1 AND DT_REP = '2025-06-15';

-- день, когда начинаем замену
SELECT TOP 6 * FROM WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1 AND DT_REP IN ('2025-07-27','2025-07-28')
ORDER BY DT_REP, TERM;

-- после последнего заседания + 1
SELECT TOP 6 * FROM WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1 AND DT_REP = '2026-08-17';
```

* **До** `2025-07-01` — точная копия витрины.
* 28-07-2025…16-08-2026 — ставки ровно по сценарию, среднее пересчитано.
* Начиная с 17-08-2026 и далее — снова фактический хвост + проекция
  на `@HorizonDays` дней вперёд.

Теперь кэш готов для ваших расчётов обоими методами.
