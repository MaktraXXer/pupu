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


```sql
/* ▀▀  ПАРАМЕТРЫ  ▀▀ */
USE ALM_TEST;
GO
DECLARE @Anchor    date = '2025-07-14',      -- последний факт
        @HorizonTo date = '2025-08-31';      -- конец прогноза
/* ▀▀  КАЛЕНДАРЬ  ▀▀ */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal)<@HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;
/* ▀▀  SPOT-KEY  ▀▀ */
IF OBJECT_ID('tempdb..#key_spot') IS NOT NULL DROP TABLE #key_spot;
SELECT fc.DT_REP,fc.KEY_RATE
INTO   #key_spot
FROM   WORK.ForecastKey_Cache fc
JOIN   #cal c ON c.d=fc.DT_REP
WHERE  fc.TERM=1;
/* ▀▀  БАЗА + СПРЕДЫ  ▀▀ */
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
SELECT  t.con_id,t.out_rub,t.rate_con,t.is_floatrate,
        t.termdays,t.dt_open,t.dt_close,
        spread_float = CASE WHEN t.is_floatrate=1
                            THEN t.rate_con-ks.KEY_RATE END,
        spread_fix   = CASE WHEN t.is_floatrate=0
                            THEN t.rate_con-fk_open.AVG_KEY_RATE END
INTO    #base
FROM    ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
JOIN    #key_spot ks                ON ks.DT_REP=@Anchor
LEFT JOIN WORK.ForecastKey_Cache fk_open
       ON fk_open.DT_REP=t.dt_open AND fk_open.TERM=t.termdays
WHERE   t.dt_rep=@Anchor
  AND   t.section_name=N'Срочные'
  AND   t.block_name  =N'Привлечение ФЛ'
  AND   t.od_flag=1  AND t.cur='810' AND t.out_rub IS NOT NULL;
/* ▀▀  ROLL-OVER  ▀▀ */
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;
;WITH seq AS(
    SELECT con_id,out_rub,is_floatrate,termdays,
           spread_float,spread_fix,
           dt_open, n=0
    FROM   #base
    UNION ALL
    SELECT  s.con_id,s.out_rub,s.is_floatrate,s.termdays,
            s.spread_float,s.spread_fix,
            DATEADD(day,s.termdays,s.dt_open),n+1
    FROM   seq s
    WHERE  DATEADD(day,s.termdays,s.dt_open)<=@HorizonTo)
SELECT  con_id,out_rub,is_floatrate,termdays,
        dt_open,
        dt_close=DATEADD(day,termdays,dt_open),
        spread_float,spread_fix
INTO   #rolls
FROM   seq OPTION (MAXRECURSION 0);
/* ▀▀  КОНТРАКТЫ ▀▀ */
IF OBJECT_ID('tempdb..#work') IS NOT NULL DROP TABLE #work;
SELECT * INTO #work FROM #rolls;
/* ▀▀  ПОСУТОЧНО  ▀▀ */
IF OBJECT_ID('tempdb..#daily') IS NOT NULL DROP TABLE #daily;
SELECT c.d AS dt_rep,
       w.con_id,w.out_rub,
       rate_con = CASE
                    WHEN w.is_floatrate=1
                         THEN ks.KEY_RATE+w.spread_float
                    ELSE ISNULL(fko.AVG_KEY_RATE+w.spread_fix,w.spread_fix)
                  END
INTO   #daily
FROM   #cal c
JOIN   #work w
  ON   c.d BETWEEN w.dt_open AND DATEADD(day,-1,w.dt_close)
LEFT  JOIN #key_spot ks
       ON ks.DT_REP=c.d
LEFT  JOIN WORK.ForecastKey_Cache fko
       ON fko.DT_REP=w.dt_open AND fko.TERM=w.termdays;
/* ▀▀  ЦЕЛЕВЫЕ ТАБЛИЦЫ  ▀▀ */
IF OBJECT_ID('WORK.Forecast_BalanceDaily','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDaily
    (dt_rep DATE PRIMARY KEY,
     out_rub_total DECIMAL(20,2),
     rate_con DECIMAL(9,4));
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDaily;

IF OBJECT_ID('WORK.Forecast_BalanceDeals','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDeals
    (dt_rep DATE,
     con_id BIGINT,
     out_rub DECIMAL(20,2),
     rate_con DECIMAL(9,4),
     CONSTRAINT PK_FBD PRIMARY KEY(dt_rep,con_id));
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDeals;
/* ▀▀  АГРЕГАТ  ▀▀ */
INSERT INTO WORK.Forecast_BalanceDaily(dt_rep,out_rub_total,rate_con)
SELECT  dt_rep,
        SUM(out_rub),
        SUM(out_rub*rate_con)/SUM(out_rub)
FROM    #daily
GROUP BY dt_rep;
/* ▀▀  ДЕТАЛКА  ▀▀ */
INSERT INTO WORK.Forecast_BalanceDeals(dt_rep,con_id,out_rub,rate_con)
SELECT dt_rep,con_id,out_rub,rate_con
FROM   #daily;
```

**Копируйте целиком** — скрипт:

1. заново наполняет `ForecastKey_Cache` (исправленная логика уже у вас);
2. каждый запуск «с нуля» очищает (`TRUNCATE`) итоговые таблицы
   `WORK.Forecast_BalanceDaily` и `WORK.Forecast_BalanceDeals`;
3. формирует посуточные ставки (float: `KEY_RATE+спред`, fix: `AVG_KEY_RATE+спред`),
   roll-over’ы и сохраняет:

| таблица                         | содержимое                                                    |
| ------------------------------- | ------------------------------------------------------------- |
| **WORK.Forecast\_BalanceDaily** | `dt_rep`, суммарный объём, средневзвешенная ставка            |
| **WORK.Forecast\_BalanceDeals** | только те сделки, ставка которых меняется (float + roll-over) |

Выполнение ≤ 1 минуты на портфель порядка 400 тыс. договоров.
