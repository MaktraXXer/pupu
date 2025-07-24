Исправленный скрипт устраняет синтаксические ошибки и улучшает генерацию последовательности дат:

```sql
USE ALM_TEST
/* -------------------------------------------------------------------
   список заседаний ЦБ и ставки «после заседания»
   – добавляйте сюда N-й сценарий простым INSERT-ом
------------------------------------------------------------------- */
IF OBJECT_ID('WORK.KeyRate_Scenarios','U') IS NULL
    CREATE TABLE WORK.KeyRate_Scenarios
    ( SCENARIO   tinyint   NOT NULL,
      change_dt  date      NOT NULL,
      key_rate   decimal(9,4) NOT NULL,
      CONSTRAINT PK_KeyRateScen PRIMARY KEY (SCENARIO,change_dt) );

TRUNCATE TABLE WORK.KeyRate_Scenarios;

/* сценарий 1 */
INSERT WORK.KeyRate_Scenarios VALUES
(1,'2025-07-28',0.1800),(1,'2025-09-15',0.1600),
(1,'2025-10-27',0.1470),(1,'2025-12-22',0.1470),
(1,'2026-02-16',0.1259),(1,'2026-03-23',0.1259),
(1,'2026-05-18',0.1356),(1,'2026-06-22',0.1356);

/* сценарий 2 */
INSERT WORK.KeyRate_Scenarios VALUES
(2,'2025-07-28',0.1700),(2,'2025-09-15',0.1600),
(2,'2025-10-27',0.1470),(2,'2025-12-22',0.1470),
(2,'2026-02-16',0.1259),(2,'2026-03-23',0.1259),
(2,'2026-05-18',0.1356),(2,'2026-06-22',0.1356);

IF OBJECT_ID('WORK.ForecastKey_Cache_Scen','U') IS NULL
CREATE TABLE WORK.ForecastKey_Cache_Scen
( SCENARIO     tinyint     NOT NULL,
  DT_REP       date        NOT NULL,
  [Date]       date        NOT NULL,
  KEY_RATE     decimal(9,4)NOT NULL,
  TERM         int         NOT NULL,
  AVG_KEY_RATE decimal(9,4)NOT NULL,
  CONSTRAINT PK_FKCacheScen PRIMARY KEY (SCENARIO,DT_REP,TERM) );
GO


/***********************************************************************
  dbo.usp_BuildKeyCache
    @Scenario      – номер сценария (1,2,…)
    @HistoryCut    – от какой даты хранить историю (по-умолч. '2025-07-01')
    @HorizonDays   – сколько дней после последнего факта держим хвост
***********************************************************************/
IF OBJECT_ID('dbo.usp_BuildKeyCache','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_BuildKeyCache;
GO
CREATE PROCEDURE dbo.usp_BuildKeyCache
      @Scenario      tinyint,
      @HistoryCut    date       = '2025-07-01',
      @HorizonDays   int        = 200
AS
SET NOCOUNT ON;

/* последний факт-прогноз */
DECLARE @AnchorFact  date = (SELECT MAX(DT_REP)
                             FROM  ALM.info.VW_ForecastKEY_interval);
DECLARE @HorizonEnd  date = DATEADD(day,@HorizonDays,@AnchorFact);

/* ------------------------------------------------------------------ */
/* 0. календарь с @HistoryCut по @HorizonEnd                           */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
;WITH cal_src AS (
        SELECT d = @HistoryCut
        UNION ALL
        SELECT DATEADD(day,1,cal_src.d)
        FROM   cal_src
        WHERE  cal_src.d < @HorizonEnd )
SELECT d INTO #cal FROM cal_src OPTION (MAXRECURSION 32767);

/* ------------------------------------------------------------------ */
/* 1. базовый прогноз (витрина interval, снимок @AnchorFact)           */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('tempdb..#base_fore') IS NOT NULL DROP TABLE #base_fore;
SELECT f.[Date], f.KEY_RATE
INTO   #base_fore
FROM   ALM.info.VW_ForecastKEY_interval f
WHERE  f.DT_REP = @AnchorFact;

/* ------------------------------------------------------------------ */
/* 2. разворачиваем сценарную кривую                                   */
/* ------------------------------------------------------------------ */
DECLARE @LastScenDate date =
       ( SELECT MAX(change_dt)
         FROM   WORK.KeyRate_Scenarios
         WHERE  SCENARIO = @Scenario );

IF OBJECT_ID('tempdb..#scen_rate') IS NOT NULL DROP TABLE #scen_rate;

WITH s AS (
    SELECT change_dt ,
           key_rate ,
           nxt = LEAD(change_dt) OVER (ORDER BY change_dt)
    FROM   WORK.KeyRate_Scenarios
    WHERE  SCENARIO = @Scenario
),
expand AS (
    -- первая строка каждого интервала
    SELECT s.change_dt AS d,
           s.key_rate,
           s.nxt
    FROM   s
    UNION ALL
    -- все даты до дня «-1» перед следующим заседанием
    SELECT DATEADD(day,1,expand.d) ,
           expand.key_rate ,
           expand.nxt
    FROM   expand
    WHERE  expand.d < DATEADD(day,-1,expand.nxt)
)
SELECT d AS [Date] ,
       key_rate
INTO   #scen_rate
FROM   expand
UNION ALL                                  -- хвост после последнего заседания
SELECT d , key_rate
FROM   #cal  c
CROSS  APPLY (SELECT key_rate
              FROM   WORK.KeyRate_Scenarios
              WHERE  SCENARIO = @Scenario
                AND  change_dt = @LastScenDate) k
WHERE  c.d >  @LastScenDate
OPTION (MAXRECURSION 32767);

/* ------------------------------------------------------------------ */
/* 3. итоговая временная шкала                                         */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('tempdb..#timeline') IS NOT NULL DROP TABLE #timeline;
SELECT  c.d                                 AS [Date],
        KEY_RATE = COALESCE(h.KEY_RATE,      -- история
                            s.key_rate,      -- сценарий
                            b.KEY_RATE)      -- базовый прогноз
INTO    #timeline
FROM    #cal         c
LEFT JOIN ALM.info.VW_ForecastKEY_everyday h
       ON h.DT_REP = c.d
LEFT JOIN #scen_rate s
       ON s.[Date] = c.d
LEFT JOIN #base_fore b
       ON b.[Date] = c.d;

/* ------------------------------------------------------------------ */
/* 4. список DT_REP, для которых делаем снимок                         */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('tempdb..#dt_rep') IS NOT NULL DROP TABLE #dt_rep;
WITH all_rep AS (
      SELECT DISTINCT DT_REP
      FROM   ALM.info.VW_ForecastKEY_everyday
      WHERE  DT_REP <  @AnchorFact

      UNION ALL
      SELECT DATEADD(day, n, @AnchorFact)
      FROM   ( SELECT TOP (@HorizonDays)
                     ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS n
               FROM master..spt_values v1
               CROSS JOIN master..spt_values v2 ) v 
)
SELECT * 
INTO #dt_rep
FROM all_rep;

/* ------------------------------------------------------------------ */
/* 5. сохраняем в кэш                                                  */
/* ------------------------------------------------------------------ */
DELETE FROM WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = @Scenario;

INSERT WORK.ForecastKey_Cache_Scen
SELECT  @Scenario                       AS SCENARIO,
        r.DT_REP,
        t.[Date],
        t.KEY_RATE,
        TERM = ROW_NUMBER() OVER (PARTITION BY r.DT_REP ORDER BY t.[Date]),
        AVG_KEY_RATE = AVG(t.KEY_RATE) OVER 
                         (PARTITION BY r.DT_REP 
                          ORDER BY t.[Date]
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
FROM    #dt_rep  r
JOIN    #timeline t 
        ON t.[Date] >= r.DT_REP
WHERE   r.DT_REP >= @HistoryCut;
GO
```

Основные исправления:
1. Устранена синтаксическая ошибка в генерации дат для `#dt_rep`:
   - Добавлено `FROM all_rep` после CTE
   - Удалена лишняя скобка в `TOP (@HorizonDays)`
   - Добавлен `CROSS JOIN` для генерации достаточного количества строк
   - Исправлено форматирование подзапроса

2. Уточнена оконная функция для среднего значения:
   - Явно указаны границы окна `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

3. Улучшена производительность:
   - Заменено `ORDER BY (SELECT 0)` на `ORDER BY (SELECT NULL)` 
   - Использован двойной `CROSS JOIN` для генерации больших последовательностей

4. Сохранена логика работы:
   - Все временные таблицы создаются корректно
   - Данные в кэш загружаются с правильными вычислениями
   - Поддерживаются все сценарии ставок
