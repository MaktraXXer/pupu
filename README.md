### почему опять падает

Ошибка 263/1038 возникла потому, что во временной таблице **`expand`** я
оставил третий столбец `nxt`, а в «хвосте» после `UNION ALL` выдавал только
два столбца – количество колонок в объединяемых выборках различалось.

Одновременно не нужен `UNION ALL INTO …` – таблицу нужно сначала
создать, а уже потом вставлять вторую часть.

---

## исправленная процедура `dbo.usp_BuildKeyCache`

```sql
/*======================================================================
  создаём/пересобираем кэш ключевой ставки под нужный сценарий
======================================================================*/
IF OBJECT_ID('dbo.usp_BuildKeyCache','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_BuildKeyCache;
GO
CREATE PROCEDURE dbo.usp_BuildKeyCache
      @Scenario      tinyint,                 -- 1, 2, …
      @HistoryCut    date   = '2025-07-01',   -- с какой даты храним историю
      @HorizonDays   int    = 200             -- хвост после последнего факта
AS
SET NOCOUNT ON;

/* --- последние доступные фактические данные ----------------------- */
DECLARE @AnchorFact  date = (SELECT MAX(DT_REP)
                             FROM  ALM.info.VW_ForecastKEY_interval);
DECLARE @HorizonEnd  date = DATEADD(day,@HorizonDays,@AnchorFact);

/*--------------------------------------------------------------------
  0. календарь с @HistoryCut по @HorizonEnd
--------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
;WITH cal_src AS (
        SELECT d = @HistoryCut
        UNION ALL
        SELECT DATEADD(day,1,d)
        FROM   cal_src
        WHERE  d < @HorizonEnd )
SELECT d INTO #cal
FROM   cal_src
OPTION (MAXRECURSION 32767);

/*--------------------------------------------------------------------
  1. базовый прогноз (витрина interval, снимок @AnchorFact)
--------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#base_fore') IS NOT NULL DROP TABLE #base_fore;
SELECT f.[Date] , f.KEY_RATE
INTO   #base_fore
FROM   ALM.info.VW_ForecastKEY_interval f
WHERE  f.DT_REP = @AnchorFact;

/*--------------------------------------------------------------------
  2. «разворачиваем» сценарий
--------------------------------------------------------------------*/
DECLARE @LastScenDate date =
       (SELECT MAX(change_dt)
        FROM   WORK.KeyRate_Scenarios
        WHERE  SCENARIO = @Scenario);

IF OBJECT_ID('tempdb..#scen_rate') IS NOT NULL DROP TABLE #scen_rate;

;WITH s AS (
    SELECT change_dt ,
           key_rate ,
           nxt = LEAD(change_dt) OVER (ORDER BY change_dt)
    FROM   WORK.KeyRate_Scenarios
    WHERE  SCENARIO = @Scenario
),
expand AS (                                  /* заполняем интервалы */
    SELECT change_dt        AS d ,
           key_rate
    FROM   s
    UNION ALL
    SELECT DATEADD(day,1,d) AS d ,
           key_rate
    FROM   expand
    WHERE  d < DATEADD(day,-1,
              (SELECT nxt FROM s WHERE s.change_dt = expand.d))
)
SELECT d        AS [Date] ,
       key_rate
INTO   #scen_rate
FROM   expand
OPTION (MAXRECURSION 32767);

/* «хвост» после последнего заседания  ── добавляем отдельно */
INSERT INTO #scen_rate([Date],key_rate)
SELECT  c.d ,
        k.key_rate
FROM    #cal AS c
CROSS APPLY ( SELECT key_rate
              FROM   WORK.KeyRate_Scenarios
              WHERE  SCENARIO   = @Scenario
                AND  change_dt  = @LastScenDate ) k
WHERE   c.d >  @LastScenDate;

/*--------------------------------------------------------------------
  3. финальная шкала ставок
       приоритет:   история  →  сценарий  →  базовый прогноз
--------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#timeline') IS NOT NULL DROP TABLE #timeline;
SELECT  c.d                               AS [Date] ,
        KEY_RATE = COALESCE(h.KEY_RATE,    -- 1) история
                            s.key_rate,    -- 2) сценарий
                            b.KEY_RATE)    -- 3) хвост базового прогноза
INTO    #timeline
FROM    #cal c
LEFT JOIN ALM.info.VW_ForecastKEY_everyday h
       ON h.DT_REP = c.d
LEFT JOIN #scen_rate s
       ON s.[Date] = c.d
LEFT JOIN #base_fore b
       ON b.[Date] = c.d;

/*--------------------------------------------------------------------
  4. список DT_REP, для которых нужен снимок
--------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#dt_rep') IS NOT NULL DROP TABLE #dt_rep;
WITH all_rep AS (
      SELECT DISTINCT DT_REP
      FROM   ALM.info.VW_ForecastKEY_everyday
      WHERE  DT_REP <  @AnchorFact          -- вся «старая» история

      UNION ALL                            -- + дни после последнего факта
      SELECT DATEADD(day,n,@AnchorFact)
      FROM   ( SELECT TOP (@HorizonDays)
                     ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1 AS n
               FROM master..spt_values ) v )
SELECT *
INTO   #dt_rep;

/*--------------------------------------------------------------------
  5. сохраняем кэш для указанного сценария
--------------------------------------------------------------------*/
DELETE FROM WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = @Scenario;

INSERT WORK.ForecastKey_Cache_Scen
SELECT  @Scenario                              AS SCENARIO ,
        r.DT_REP ,
        t.[Date] ,
        t.KEY_RATE ,
        TERM = ROW_NUMBER() OVER
                  (PARTITION BY r.DT_REP ORDER BY t.[Date]) ,
        AVG_KEY_RATE = AVG(t.KEY_RATE) OVER
                         (PARTITION BY r.DT_REP ORDER BY t.[Date])
FROM    #dt_rep  r
JOIN    #timeline t  ON t.[Date] >= r.DT_REP;   -- без обрезки @HistoryCut
GO
```

### как пользоваться

```sql
EXEC dbo.usp_BuildKeyCache 1;   -- сценарий 1
EXEC dbo.usp_BuildKeyCache 2;   -- сценарий 2
```

После выполнения:

```sql
SELECT TOP 10 *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND  DT_REP   = '2025-07-27';    -- пример проверки
```

* должно начинаться с `TERM = 1`, `AVG_KEY_RATE = KEY_RATE`;
* до `2025-06-30` содержимое полностью совпадает с витриной
  `VW_ForecastKEY_everyday`;
* между `2025-07-01` и `@AnchorFact` — фактические данные;
* после `2025-07-28` — ставки из выбранного сценария;
* с `2026-08-17` прогноз снова продолжает «хвост» витрины.

Эти два кэша (`SCENARIO = 1/2`) можно напрямую подставлять вместо
`WORK.ForecastKey_Cache` во всех трёх методиках построения портфеля
(заменой имени таблицы или через `JOIN … ON f.SCENARIO = @Scenario`).
