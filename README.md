#### что нужно получить

* **Две независимые «кэширующие» таблицы**
  `WORK.ForecastKey_Cache_S1` - для сценария 1
  `WORK.ForecastKey_Cache_S2` - для сценария 2

* В них:

| диапазон `DT_REP`               | откуда берётся KEY\_RATE                                                             |
| ------------------------------- | ------------------------------------------------------------------------------------ |
| **< 2025-07-01**                | полностью из истории (`VW_ForecastKEY_everyday`)                                     |
| **2025-07-01 … @AnchorFact**    | фактический прогноз, уже лежащий в базе                                              |
| **@AnchorFact+1 … @HorizonEnd** | ключ по сценарию: значение меняется только в даты заседаний, между ними - «держится» |

* `TERM` и `AVG_KEY_RATE` пересчитываются **так же, как в вашем исходном кэше**
  (т.е. для каждого `DT_REP` «хвост» прогноза начинается с `TERM = 1`,
  а `AVG_KEY_RATE` — среднее от 1-го срока до текущего).

---

## 1 — каркас: задаём сценарии

```sql
/* ─── СЦЕНАРИИ КЛЮЧЕВОЙ СТАВКИ: что меняется и когда ─── */
IF OBJECT_ID('tempdb..#scenario_rate') IS NOT NULL DROP TABLE #scenario_rate;
CREATE TABLE #scenario_rate
( scenario tinyint,  -- 1 или 2
  change_dt date,   -- дата заседания ЦБ
  key_rate  decimal(9,4) );

/* сценарий 1 */
INSERT #scenario_rate VALUES
(1,'2025-07-28',0.1800),(1,'2025-09-15',0.1600),
(1,'2025-10-27',0.1470),(1,'2025-12-22',0.1470),
(1,'2026-02-16',0.1259),(1,'2026-03-23',0.1259),
(1,'2026-05-18',0.1259),(1,'2026-06-22',0.1259);

/* сценарий 2 */
INSERT #scenario_rate VALUES
(2,'2025-07-28',0.1700),(2,'2025-09-15',0.1600),
(2,'2025-10-27',0.1470),(2,'2025-12-22',0.1470),
(2,'2026-02-16',0.1259),(2,'2026-03-23',0.1259),
(2,'2026-05-18',0.1259),(2,'2026-06-22',0.1259);
```

> **Если нужно новый сценарий** — просто добавьте строки
> `INSERT #scenario_rate VALUES (3,'2025-08-31',0.1750)…`

---

## 2 — один скрипт, который собирает нужный кэш

```sql
/***********************************************************************
  proc dbo.usp_BuildKeyCacheScenario
    @Scenario tinyint        -- 1 / 2 / …
    ,@Horizon int = 200      -- сколько дней вперёд считаем
***********************************************************************/
IF OBJECT_ID('dbo.usp_BuildKeyCacheScenario','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_BuildKeyCacheScenario;
GO
CREATE PROCEDURE dbo.usp_BuildKeyCacheScenario
      @Scenario tinyint
    , @Horizon  int = 200
AS
SET NOCOUNT ON;

/* ---- исходные даты ---- */
DECLARE @AnchorFact date = (SELECT MAX(DT_REP)          -- t-1
                            FROM  ALM.info.VW_ForecastKEY_interval),
        @HistoryCut date = '2025-07-01',                -- где «обрываем» историю
        @HorizonEnd date = DATEADD(day,@Horizon,@AnchorFact);

/* ---- 0. Timeline всех нужных дат и ставок ---- */
IF OBJECT_ID('tempdb..#timeline') IS NOT NULL DROP TABLE #timeline;
WITH d AS (
    SELECT DATEADD(day,0,@HistoryCut) AS d
    UNION ALL
    SELECT DATEADD(day,1,d) FROM d WHERE d < @HorizonEnd )
SELECT  d.d                                  AS [Date],
        /* историческую часть берём из витрины,
           будущее — по сценарию                                   */
        key_rate = COALESCE(h.key_rate,
                            /* «последняя» ставка сценария ≤ текущей даты */
                            ( SELECT TOP 1 s.key_rate
                              FROM   #scenario_rate s
                              WHERE  s.scenario = @Scenario
                                AND  s.change_dt <= d.d
                              ORDER  BY s.change_dt DESC ))
INTO    #timeline
FROM    d
LEFT    JOIN ALM.info.VW_ForecastKEY_interval h
           ON h.DT_REP = @AnchorFact AND h.[Date] = d.d
OPTION (MAXRECURSION 32767);   -- нужно, если горизонт > 32767

/* ---- 1. заготовка дат DT_REP, для которых сделаем «снимок» ---- */
IF OBJECT_ID('tempdb..#dt_rep') IS NOT NULL DROP TABLE #dt_rep;
WITH a AS (
    /* история до AnchorFact ― как в everyday-витрине */
    SELECT DISTINCT DT_REP
    FROM   ALM.info.VW_ForecastKEY_everyday
    WHERE  DT_REP <  @AnchorFact

    UNION ALL
    /* плюс будущие dt_rep от (AnchorFact+1) до HorizonEnd */
    SELECT DATEADD(day,v.n,@AnchorFact)
    FROM  (SELECT TOP (DATEDIFF(day,0,@HorizonEnd)) ROW_NUMBER() OVER(ORDER BY (SELECT 0))-1 AS n
           FROM master..spt_values) v
    WHERE v.n BETWEEN 1 AND @Horizon
)
SELECT * INTO #dt_rep;

/* ---- 2. собираем финальный кэш ---- */
DECLARE @tblName sysname = CONCAT('WORK.ForecastKey_Cache_S',@Scenario);

EXEC('
IF OBJECT_ID('''+@tblName+''',''U'') IS NOT NULL DROP TABLE '+@tblName+';
CREATE TABLE '+@tblName+'
( DT_REP date NOT NULL,
  [Date] date NOT NULL,
  KEY_RATE decimal(9,4) NOT NULL,
  TERM int NOT NULL,
  AVG_KEY_RATE decimal(9,4) NOT NULL,
  CONSTRAINT PK_'+@tblName+' PRIMARY KEY CLUSTERED (DT_REP,TERM) );');

/* 2-а. история (< AnchorFact) — как есть */
INSERT '+@tblName+'
SELECT DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE
FROM   ALM.info.VW_ForecastKEY_everyday
WHERE  DT_REP < @AnchorFact
  AND  DT_REP >= @HistoryCut;

/* 2-б. снимки от AnchorFact и дальше */
INSERT '+@tblName+'
SELECT  r.DT_REP,
        t.[Date],
        t.key_rate,
        TERM = ROW_NUMBER() OVER (PARTITION BY r.DT_REP ORDER BY t.[Date]),
        AVG_KEY_RATE = AVG(t.key_rate) OVER (PARTITION BY r.DT_REP ORDER BY t.[Date])
FROM    #dt_rep  r
JOIN    #timeline t
      ON t.[Date] >= r.DT_REP
WHERE   r.DT_REP >= @AnchorFact;');
GO
```

*Вызываем так:*

```sql
EXEC dbo.usp_BuildKeyCacheScenario @Scenario = 1;  -- сделает WORK.ForecastKey_Cache_S1
EXEC dbo.usp_BuildKeyCacheScenario @Scenario = 2;  -- сделает WORK.ForecastKey_Cache_S2
```

> **Если горизонт нужен другой** — добавьте `,@Horizon = 300` при вызове.

---

## 3 — как подключить в ваши три методики

* В каждом калькуляторе ставок **замените** строку

```sql
FROM   WORK.ForecastKey_Cache fc   -- было
```

на

```sql
FROM   WORK.ForecastKey_Cache_S1 fc   -- или _S2
```

* Больше менять ничего не нужно: структура таблицы полностью та же,
  `TERM` начинается с 1, `AVG_KEY_RATE` корректный.

---

### Проверка себя

```sql
/* убедимся, что TERM и AVG_KEY_RATE идут правильно */
SELECT TOP 10 *
FROM   WORK.ForecastKey_Cache_S1
WHERE  DT_REP = '2025-07-29';

/* сравним сценарии между собой */
SELECT  a.[Date],
        s1 = s1.KEY_RATE,
        s2 = s2.KEY_RATE
FROM    WORK.ForecastKey_Cache_S1 s1
JOIN    WORK.ForecastKey_Cache_S2 s2
          ON s2.DT_REP = s1.DT_REP AND s2.[Date] = s1.[Date]
WHERE   s1.DT_REP = '2025-07-29'
  AND   s1.KEY_RATE <> s2.KEY_RATE;
```

*В диапазоне 28-07 – 14-09-2025 вы увидите разницу: 18 % против 17 %.*
Все остальные даты совпадут, что и требовалось.

---

> В итоге у вас есть **универсальный генератор кэша**.
> Хотите новую траекторию ключевой ставки — добавляете сценарий в `#scenario_rate` и вызываете процедуру с нужным id.
