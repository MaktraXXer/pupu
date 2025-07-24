ниже — полный, «самодостаточный» текст процедуры **dbo.usp\_BuildKeyCache**.
Я убрал все дублирующиеся объявления и оставил только один блок `DECLARE`, а дальше — строгое следование порядку «сначала объявили → потом присвоили → потом использовали».

```sql
USE ALM_TEST;
GO
IF OBJECT_ID('dbo.usp_BuildKeyCache','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_BuildKeyCache;
GO
/***********************************************************************
  dbo.usp_BuildKeyCache
     @Scenario    – номер сценария из WORK.KeyRate_Scenarios
     @HistoryCut  – с какой даты начинаем применять сценарий
     @HorizonDays – сколько дней после t-1 держим хвост
***********************************************************************/
CREATE PROCEDURE dbo.usp_BuildKeyCache
      @Scenario     tinyint,                 -- 1 / 2 / 3 …
      @HistoryCut   date       = '2025-07-01',
      @HorizonDays  int        = 200
AS
SET NOCOUNT ON;

/*--------------------------------------------------------------------
  0. опорные даты и «таблица чисел»
--------------------------------------------------------------------*/
DECLARE
    @AnchorFact     date,                    -- t-1 (последний фактический DT_REP)
    @ScenCutOff     date   = '2026-08-16',   -- день перед след. заседанием
    @LastSnapDate   date;                    -- @AnchorFact + @HorizonDays

SELECT @AnchorFact = MAX(DT_REP)
FROM   ALM.info.VW_ForecastKEY_interval;

SET @LastSnapDate = DATEADD(day, @HorizonDays, @AnchorFact);

IF OBJECT_ID('tempdb..#nums') IS NOT NULL DROP TABLE #nums;
;WITH seq AS (
    SELECT TOP (DATEDIFF(day,'2000-01-01','2030-12-31')+1)
           n = ROW_NUMBER() OVER(ORDER BY (SELECT NULL))-1
    FROM   sys.all_objects a CROSS JOIN sys.all_objects b
)
SELECT  n,
        d = DATEADD(day,n,'2000-01-01')
INTO    #nums;

/*--------------------------------------------------------------------
  1. чищу кэш сценария и переношу всю «старую» историю (< @HistoryCut)
--------------------------------------------------------------------*/
DELETE FROM WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = @Scenario;

INSERT INTO WORK.ForecastKey_Cache_Scen
        (SCENARIO, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE)
SELECT  @Scenario,
        DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
FROM    ALM.info.VW_ForecastKEY_everyday
WHERE   DT_REP < @HistoryCut;               -- до правки ничего не меняем

/*--------------------------------------------------------------------
  2. формирую «день-в-день» ставку из сценария (FirstScen…@ScenCutOff)
--------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#scen_day') IS NOT NULL DROP TABLE #scen_day;

;WITH borders AS (
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
      FROM   borders b
      JOIN   #nums n
             ON n.d BETWEEN b.change_dt
                        AND DATEADD(day,-1,ISNULL(b.nxt,@ScenCutOff))
)
SELECT [Date], key_rate
INTO   #scen_day
FROM   days;

/*--------------------------------------------------------------------
  3. общий тайм-лайн от @HistoryCut до @LastSnapDate
--------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#timeline') IS NOT NULL DROP TABLE #timeline;

;WITH cal AS (
      SELECT d
      FROM   #nums
      WHERE  d BETWEEN @HistoryCut AND @LastSnapDate
),
hist_one_day AS (
      SELECT DT_REP AS d, KEY_RATE
      FROM   ALM.info.VW_ForecastKEY_everyday
      WHERE  DT_REP BETWEEN @HistoryCut AND @AnchorFact
),
fore_t1 AS (
      SELECT [Date] AS d, KEY_RATE
      FROM   ALM.info.VW_ForecastKEY_interval
      WHERE  DT_REP = @AnchorFact
        AND  [Date] BETWEEN @AnchorFact AND @LastSnapDate
)
SELECT  c.d                   AS [Date],
        KEY_RATE = COALESCE(s.key_rate,
                            h.KEY_RATE,
                            f.KEY_RATE)
INTO    #timeline
FROM    cal         c
LEFT    JOIN #scen_day    s ON s.[Date] = c.d
LEFT    JOIN hist_one_day h ON h.d      = c.d
LEFT    JOIN fore_t1      f ON f.d      = c.d;

/*--------------------------------------------------------------------
  4. заполняю кэш: @HistoryCut … @LastSnapDate
--------------------------------------------------------------------*/
;WITH all_rep AS (
      SELECT d
      FROM   #nums
      WHERE  d BETWEEN @HistoryCut AND @LastSnapDate
)
INSERT INTO WORK.ForecastKey_Cache_Scen
        (SCENARIO, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE)
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
FROM   all_rep r
JOIN   #timeline t ON t.[Date] >= r.d;   -- «хвост» от DT_REP и далее
GO
```

### Проверка

```sql
EXEC dbo.usp_BuildKeyCache @Scenario = 1;

-- день до первого заседания
SELECT TOP (5) *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1 AND DT_REP = '2025-07-24'
ORDER  BY TERM;

-- должна быть ставка 0.18 (18 %) с 28-го июля
SELECT *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND  DT_REP   = '2025-07-24'
  AND  [Date]   BETWEEN '2025-07-27' AND '2025-07-29';
```

Если вывод совпадает с ожиданиями (с 28-го 07 2025 ставка 18 %), то процедура работает корректно.
Запустите аналогично для `@Scenario = 2` и `3`, чтобы заполнить все версии кэша.
