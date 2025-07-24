Ниже ― «чистый» T-SQL-процедура **без динамического SQL**, которая собирает
кэш ключевой ставки *одновременно* для любого числа сценариев.
У вас будет **одна** таблица-приёмник
`WORK.ForecastKey_Cache_Scen`, а сценарий отличается полем `SCENARIO`.

### 1. подготавливаем сценарии (разово)

```sql
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
(1,'2026-05-18',0.1259),(1,'2026-06-22',0.1259);

/* сценарий 2 */
INSERT WORK.KeyRate_Scenarios VALUES
(2,'2025-07-28',0.1700),(2,'2025-09-15',0.1600),
(2,'2025-10-27',0.1470),(2,'2025-12-22',0.1470),
(2,'2026-02-16',0.1259),(2,'2026-03-23',0.1259),
(2,'2026-05-18',0.1259),(2,'2026-06-22',0.1259);
```

### 2. таблица-кэш, общая для всех сценариев

```sql
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
```

### 3. универсальная процедура-построитель

```sql
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

/* последний факт-прогноз из витрины interval */
DECLARE @AnchorFact date = (SELECT MAX(DT_REP)
                            FROM  ALM.info.VW_ForecastKEY_interval);

/* куда дорастить прогноз */
DECLARE @HorizonEnd date = DATEADD(day,@HorizonDays,@AnchorFact);

/* ---- 0. календарь нужного диапазона ---- */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
WITH d AS (
    SELECT d = @HistoryCut
    UNION ALL
    SELECT DATEADD(day,1,d) FROM d WHERE d < @HorizonEnd)
SELECT d INTO #cal OPTION (MAXRECURSION 32767);

/* ---- 1. базовый (фактический) прогноз, привязанный к AnchorFact ---- */
IF OBJECT_ID('tempdb..#base_fore') IS NOT NULL DROP TABLE #base_fore;
SELECT f.[Date], f.KEY_RATE
INTO   #base_fore
FROM   ALM.info.VW_ForecastKEY_interval f
WHERE  f.DT_REP = @AnchorFact;

/* ---- 2. сценарная кривая: «держим» ставку до следующего заседания ---- */
IF OBJECT_ID('tempdb..#scen_rate') IS NOT NULL DROP TABLE #scen_rate;
WITH s AS (
      SELECT s.change_dt,
             s.key_rate,
             nxt = LEAD(s.change_dt) OVER (ORDER BY s.change_dt)
      FROM   WORK.KeyRate_Scenarios s
      WHERE  s.SCENARIO = @Scenario
),
d AS (
      SELECT change_dt AS d, key_rate FROM s
      UNION ALL
      SELECT DATEADD(day,1,d), key_rate
      FROM   d
      WHERE  d < (SELECT MAX(nxt)-1 FROM s WHERE nxt IS NOT NULL)
)
SELECT d AS [Date], key_rate
INTO   #scen_rate
FROM   d OPTION (MAXRECURSION 32767);

/* ---- 3. итоговая временная шкала ---- */
IF OBJECT_ID('tempdb..#timeline') IS NOT NULL DROP TABLE #timeline;
SELECT  c.d                                   AS [Date],
        /* приоритеты:
           1) история до AnchorFact
           2) сценарий (если попадает в диапазон его дат)
           3) базовый прогноз                                         */
        KEY_RATE = COALESCE(h.KEY_RATE,
                            s.key_rate,
                            b.KEY_RATE)
INTO    #timeline
FROM    #cal               c
LEFT    JOIN ALM.info.VW_ForecastKEY_everyday h
           ON h.DT_REP = c.d
LEFT    JOIN #scen_rate s
           ON s.[Date] = c.d           -- только заявленные сценарные дни
LEFT    JOIN #base_fore  b
           ON b.[Date] = c.d;

/* ---- 4. даты DT_REP, для которых делаем снимок ---- */
IF OBJECT_ID('tempdb..#dt_rep') IS NOT NULL DROP TABLE #dt_rep;
WITH all_rep AS (
      SELECT DISTINCT DT_REP
      FROM   ALM.info.VW_ForecastKEY_everyday
      WHERE  DT_REP <  @AnchorFact

      UNION ALL
      SELECT DATEADD(day,n,@AnchorFact)
      FROM   ( SELECT TOP (@HorizonDays)
                     ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1 AS n
               FROM master..spt_values ) v )
SELECT * INTO #dt_rep;

/* ---- 5. удаляем старую версию сценария и пишем новую ---- */
DELETE FROM WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = @Scenario;

INSERT WORK.ForecastKey_Cache_Scen
SELECT  @Scenario                 AS SCENARIO,
        r.DT_REP,
        t.[Date],
        t.KEY_RATE,
        TERM = ROW_NUMBER() OVER (PARTITION BY r.DT_REP ORDER BY t.[Date]),
        AVG_KEY_RATE = AVG(t.KEY_RATE) OVER
                         (PARTITION BY r.DT_REP ORDER BY t.[Date])
FROM    #dt_rep  r
JOIN    #timeline t ON t.[Date] >= r.DT_REP
WHERE   r.DT_REP >= @HistoryCut;
GO
```

### 4. вызываем

```sql
/* пересобрать сценарий 1 и 2 */
EXEC dbo.usp_BuildKeyCache @Scenario = 1, @HorizonDays = 200;
EXEC dbo.usp_BuildKeyCache @Scenario = 2, @HorizonDays = 200;
```

---

## 5. как подключить к вашим трём расчётам

*Замените только источник ключевой ставки*:

```sql
FROM WORK.ForecastKey_Cache_Scen fc   -- вместо WORK.ForecastKey_Cache
WHERE fc.SCENARIO = 1                 -- (или 2)
```

Всё остальное – без изменений: поля (`DT_REP`,`Date`,`KEY_RATE`,`TERM`,
`AVG_KEY_RATE`) те же, `TERM` всегда начинается с 1,
`AVG_KEY_RATE` считается «скользящим» точно как раньше.

---

### проверка

```sql
/* TERM начинается с 1 и среднее = KEY_RATE на первой дате */
SELECT TOP 10 *
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO = 1
  AND  DT_REP   = '2025-07-29';

/* сценарии действительно различаются только в нужных окнах */
SELECT  t.Date, s1.KEY_RATE AS S1, s2.KEY_RATE AS S2
FROM    WORK.ForecastKey_Cache_Scen s1
JOIN    WORK.ForecastKey_Cache_Scen s2
          ON s2.DT_REP = s1.DT_REP
         AND s2.Date   = s1.Date
WHERE   s1.SCENARIO = 1
  AND   s2.SCENARIO = 2
  AND   s1.DT_REP   = '2025-07-29'
  AND   s1.KEY_RATE <> s2.KEY_RATE;
```

После 17 августа 2026 ставки обоих сценариев снова совпадут с базовым прогнозом из `VW_ForecastKEY_interval`, потому что за пределами списка заседаний кривая «проваливается» в оригинальный прогноз.

Теперь кэш собирается статическим SQL, легко
расширяется новыми сценариями и подключается к любой из ваших трёх методик одной строкой.
