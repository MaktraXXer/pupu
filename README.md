Вот цельная, компактная версия процедуры — без единого скрытого `GO`, с по-шаговым
инициализационным блоком (`DECLARE … ; SET …`) — чтобы SQL Server не «терял» переменные
при компиляции.

```sql
USE ALM_TEST;
GO
IF OBJECT_ID('dbo.usp_BuildKeyCache','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_BuildKeyCache;
GO
CREATE PROCEDURE dbo.usp_BuildKeyCache
      @Scenario      tinyint,                 -- 1 / 2 / 3 …
      @HistoryCut    date  = '2025-07-01',    -- откуда начинаем править
      @HorizonDays   int   = 200              -- хвост после t-1
AS
BEGIN
    SET NOCOUNT ON;

    /*───────────────────────── 0. опорные даты ─────────────────────*/
    DECLARE
        @AnchorFact   date,
        @ScenCutOff   date = '2026-08-16',          -- день перед заседанием
        @LastSnapDate date;                         -- t-1 + хвост

    SELECT @AnchorFact = MAX(DT_REP)
    FROM   ALM.info.VW_ForecastKEY_interval;        -- t-1

    SET @LastSnapDate = DATEADD(day,@HorizonDays,@AnchorFact);

    /*────────────── 1. таблица чисел на 2000-01-01 … 2030-12-31 ─────*/
    IF OBJECT_ID('tempdb..#num') IS NOT NULL DROP TABLE #num;
    ;WITH seq AS (
         SELECT TOP (DATEDIFF(day,'2000-01-01','2030-12-31')+1)
                rn = ROW_NUMBER() OVER (ORDER BY (SELECT NULL))-1
         FROM sys.all_objects a CROSS JOIN sys.all_objects b
    )
    SELECT n = rn,
           d = DATEADD(day,rn,'2000-01-01')
    INTO   #num
    FROM   seq
    OPTION (MAXRECURSION 0);

    /*────────────── 2. «доправочную» историю копируем как есть ─────*/
    DELETE FROM WORK.ForecastKey_Cache_Scen
    WHERE  SCENARIO = @Scenario;

    INSERT INTO WORK.ForecastKey_Cache_Scen
    SELECT @Scenario,
           DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
    FROM   ALM.info.VW_ForecastKEY_everyday
    WHERE  DT_REP < @HistoryCut;

    /*────────────── 3. сценарий → «день-в-день» (#scen_day) ────────*/
    IF OBJECT_ID('tempdb..#scen_day') IS NOT NULL DROP TABLE #scen_day;

    ;WITH borders AS (
          SELECT change_dt,
                 key_rate,
                 nxt = LEAD(change_dt) OVER (ORDER BY change_dt)
          FROM   WORK.KeyRate_Scenarios
          WHERE  SCENARIO = @Scenario
    ),
    spread AS (
          SELECT DATEADD(day,n.n,b.change_dt) AS [Date],
                 b.key_rate
          FROM   borders b
          JOIN   #num n
                 ON n.d BETWEEN b.change_dt
                            AND DATEADD(day,-1,ISNULL(b.nxt,@ScenCutOff))
    )
    SELECT  [Date], key_rate
    INTO    #scen_day
    FROM    spread;

    /*────────────── 4. тайм-лайн от HistoryCut до LastSnapDate ─────*/
    IF OBJECT_ID('tempdb..#timeline') IS NOT NULL DROP TABLE #timeline;

    ;WITH calendar AS (
          SELECT d
          FROM   #num
          WHERE  d BETWEEN @HistoryCut AND @LastSnapDate
    ),
    hist AS (
          SELECT DT_REP AS d, KEY_RATE
          FROM   ALM.info.VW_ForecastKEY_everyday
          WHERE  DT_REP BETWEEN @HistoryCut AND @AnchorFact
    ),
    fore AS (
          SELECT [Date] AS d, KEY_RATE
          FROM   ALM.info.VW_ForecastKEY_interval
          WHERE  DT_REP = @AnchorFact
            AND  [Date] BETWEEN @AnchorFact AND @LastSnapDate
    )
    SELECT  c.d                    AS [Date],
            KEY_RATE = COALESCE(s.key_rate,   -- 1) сценарий
                                h.KEY_RATE,   -- 2) факт
                                f.KEY_RATE)   -- 3) прогноз t-1
    INTO    #timeline
    FROM    calendar c
    LEFT    JOIN #scen_day s ON s.[Date] = c.d
    LEFT    JOIN hist      h ON h.d      = c.d
    LEFT    JOIN fore      f ON f.d      = c.d;

    /*────────────── 5. строим кэш DT_REP ≥ HistoryCut ──────────────*/
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
    JOIN   #timeline t ON t.[Date] >= r.d;
END
GO
```

### Что изменилось / почему теперь компилируется

| Было                                                                                                           | Стало                                                                                                                          |
| -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `DECLARE …, @LastSnapDate date = DATEADD(…, @AnchorFact)` — SQL Server ругается: переменная ещё «не объявлена» | Разделил на `DECLARE` + отдельные `SET` — каждую переменную вычисляем **после** того, как получили все, на которых она зависит |
| Переменные использовались в определениях CTE ещё до объявления                                                 | Сейчас все CTE идут **после** блока `DECLARE / SET`                                                                            |
| Имя временной таблицы `#num` каждый раз создавалось заново → конфликт                                          | Перед созданием `DROP TABLE` если есть; имя единообразное                                                                      |

Проверьте быстро:

```sql
EXEC dbo.usp_BuildKeyCache @Scenario = 1;

-- Должно отразить 0.18 c 28-07-25
SELECT TOP 10 *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND  DT_REP   = '2025-07-24'
ORDER BY TERM;
```

Если нужно другие сценарии:

```sql
EXEC dbo.usp_BuildKeyCache @Scenario = 2;
EXEC dbo.usp_BuildKeyCache @Scenario = 3;
```

Каждый вызов полностью перезаписывает кэш для выбранного сценария, не затрагивая прочие.
