Ниже — корректный скрипт «с нуля»:

* **историю** берём напрямую из готовой витрины `VW_ForecastKEY_everyday` (там всё уже разложено правильно);
* **проекцию вперёд** строим на базе последнего доступного `DT_REP`, но
  обязательно сохраняем условие `f.[Date] >= DT_REP` — иначе появляются
  лишние строки и «растёт» `TERM`.

```sql
/*------------------------------------------------------------------
  0.   параметры
------------------------------------------------------------------*/
USE ALM_TEST;
GO

DECLARE
    @Anchor    date = (SELECT MAX(DT_REP)                    -- t-1
                       FROM  ALM.info.VW_ForecastKEY_interval),
    @Horizon   int  = 145;                                   -- дней вперёд

/*------------------------------------------------------------------
  1.   кэш-таблица
------------------------------------------------------------------*/
IF OBJECT_ID('WORK.ForecastKey_Cache','U') IS NOT NULL
    DROP TABLE WORK.ForecastKey_Cache;
GO

CREATE TABLE WORK.ForecastKey_Cache
( DT_REP       date         NOT NULL,
  [Date]       date         NOT NULL,
  KEY_RATE     decimal(9,4) NOT NULL,
  TERM         int          NOT NULL,
  AVG_KEY_RATE decimal(9,4) NOT NULL,
  CONSTRAINT PK_FKCache PRIMARY KEY CLUSTERED (DT_REP, TERM)
);
GO

/*------------------------------------------------------------------
  2.   История  — просто копия витрины (до @Anchor)
------------------------------------------------------------------*/
INSERT INTO WORK.ForecastKey_Cache (DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE)
SELECT DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
FROM   ALM.info.VW_ForecastKEY_everyday  WITH (NOLOCK)
WHERE  DT_REP <= @Anchor;

/*------------------------------------------------------------------
  3.   Проекция вперёд: @Anchor+1 … @Anchor+@Horizon
------------------------------------------------------------------*/
;WITH nums AS (SELECT TOP (@Horizon) ROW_NUMBER() OVER (ORDER BY (SELECT 0)) AS n
               FROM sys.all_objects)
INSERT INTO WORK.ForecastKey_Cache (DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE)
SELECT
        DT_Future       = DATEADD(day,n,@Anchor),        -- новый DT_REP
        f.[Date],
        f.KEY_RATE,
        TERM            = ROW_NUMBER() OVER
                            (PARTITION BY DATEADD(day,n,@Anchor) ORDER BY f.[Date]),
        AVG_KEY_RATE    = AVG(f.KEY_RATE) OVER
                            (PARTITION BY DATEADD(day,n,@Anchor) ORDER BY f.[Date])
FROM    nums
JOIN    ALM.info.VW_ForecastKEY_interval f  WITH (NOLOCK)
      ON f.DT_REP = @Anchor                 -- прогноз последнего снимка
     AND f.[Date] >= DATEADD(day,n,@Anchor) -- ключ всегда ≥ DT_REP
OPTION (MAXRECURSION 0);
```

### Что изменилось

| было                                                                                 | стало                                                                                                                    |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| Историю собирали «самодельным» join’ом и пропускали условие `f.[Date] >= cal.[Date]` | Берём строки **как есть** из витрины — никаких лишних дат, `TERM` начинается с 1.                                        |
| В проекции не фильтровали по `f.[Date] >= новому DT_REP`                             | Фильтр добавлен, поэтому в каждом будущем снимке остаётся только «хвост» прогноза, и `AVG_KEY_RATE` считается правильно. |

Проверьте теперь:

```sql
SELECT TOP 10 *
FROM   WORK.ForecastKey_Cache
WHERE  DT_REP = '2025-07-14';
```

должно начинаться с `TERM = 1`, `AVG_KEY_RATE = KEY_RATE` и **никаких** дат до 14-07-2025.
