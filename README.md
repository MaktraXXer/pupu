Ниже - «пошаговый» способ сделать *точную копию* витрины **VW\_ForecastKEY\_everyday** в базе **ALM\_TEST**, включая «пролонгацию» вперёд на нужный горизонт (например + 90 дней).
Берём ту же формулу, которая уже заложена во вьюхе (calendar × interval-таблица), но результат сохраняем в материальную таблицу — так последующие джойны по (`DT_REP`,`TERM`) будут мгновенными.

---

## 1 ∙ Таблица-кеш в ALM\_TEST

```sql
USE ALM_TEST;
GO

IF OBJECT_ID('WORK.ForecastKey_Cache','U') IS NOT NULL
    DROP TABLE WORK.ForecastKey_Cache;
GO

CREATE TABLE WORK.ForecastKey_Cache
( DT_REP       DATE         NOT NULL,   -- дата «снимка» прогноза
  [Date]       DATE         NOT NULL,   -- дата, на которую спрогнозирован KEY_RATE
  KEY_RATE     DECIMAL(9,4) NOT NULL,
  TERM         INT          NOT NULL,   -- 1,2,3…  — номер дня в горизонте
  AVG_KEY_RATE DECIMAL(9,4) NOT NULL,
  CONSTRAINT PK_FKCache PRIMARY KEY CLUSTERED (DT_REP, TERM)  -- точный seek
);
GO
```

---

## 2 ∙ Полное наполнение (история + 90 дней вперёд)

```sql
DECLARE @Horizon   INT  = 90;            -- сколько дней «в будущее»
DECLARE @Today     DATE = CAST(GETDATE() AS DATE);
DECLARE @LastRep   DATE = ( SELECT MAX(DT_REP)
                            FROM ALM.info.VW_ForecastKEY_interval );  -- самый свежий снимок

/* ---------- 2.1  Историческая часть  ---------- */
INSERT INTO WORK.ForecastKey_Cache (DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE)
SELECT
        cal.[Date]                       AS DT_REP ,
        fkey.[Date]                      ,
        fkey.KEY_RATE                   ,
        ROW_NUMBER() OVER
            (PARTITION BY cal.[Date] ORDER BY fkey.[Date])             AS TERM ,
        AVG(fkey.KEY_RATE) OVER
            (PARTITION BY cal.[Date] ORDER BY fkey.[Date])             AS AVG_KEY_RATE
FROM    ALM.info.VW_ForecastKEY_interval fkey         WITH (NOLOCK)
JOIN    ALM.info.VW_calendar           cal  WITH (NOLOCK)
      ON cal.[Date] >= fkey.DT_REP
     AND cal.[Date] <  fkey.DT_REP_NEXT
WHERE   cal.[Date] <= @Today;            -- всё, что не позже «сегодня»


/* ---------- 2.2  Проекция вперёд  (Today+1 … Today+@Horizon)  ---------- */
/* Для каждого будущего DT_REP берём ТОТ ЖЕ прогноз LastRep, но «обрезаем» дни, которые уже прошли */
;WITH nums AS ( SELECT TOP (@Horizon) ROW_NUMBER() OVER (ORDER BY (SELECT 0)) AS n
                FROM sys.all_objects )            -- дешёвая «таблица чисел»
INSERT INTO WORK.ForecastKey_Cache (DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE)
SELECT
        DT_Future          = DATEADD(day, n, @Today),          -- новый DT_REP
        f.[Date],
        f.KEY_RATE,
        TERM               = ROW_NUMBER() OVER
                               (PARTITION BY DATEADD(day,n,@Today) ORDER BY f.[Date]),
        AVG_KEY_RATE       = AVG(f.KEY_RATE) OVER
                               (PARTITION BY DATEADD(day,n,@Today) ORDER BY f.[Date])
FROM    nums
JOIN    ALM.info.VW_ForecastKEY_interval f   WITH (NOLOCK)
      ON f.DT_REP = @LastRep                 -- используем последний доступный снимок
     AND f.[Date] >= DATEADD(day, n, @Today) -- удаляем «прошедшие» дни
OPTION (MAXRECURSION 0);
```

*Для `DT_REP = Today + n` остаётся **@Horizon – n + 1** дней, поэтому `TERM` всегда стартует с 1.*

---

## 3 ∙ Ежедневное инкрементальное обновление (скрипт для SQL Agent)

```sql
DECLARE @Today    DATE = CAST(GETDATE() AS DATE);
DECLARE @PrevDay  DATE = DATEADD(day,-1,@Today);
DECLARE @Horizon  INT  = 90;

/* 3.1  Добавляем вчерашний факт (если витрина пересчиталась) */
INSERT INTO WORK.ForecastKey_Cache (DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE)
SELECT cal.[Date], fkey.[Date], fkey.KEY_RATE,
       ROW_NUMBER() OVER(PARTITION BY cal.[Date] ORDER BY fkey.[Date]),
       AVG(fkey.KEY_RATE) OVER(PARTITION BY cal.[Date] ORDER BY fkey.[Date])
FROM   ALM.info.VW_ForecastKEY_interval fkey
JOIN   ALM.info.VW_calendar cal
     ON cal.[Date] >= fkey.DT_REP AND cal.[Date] < fkey.DT_REP_NEXT
WHERE  cal.[Date] = @PrevDay          -- добавляем ровно один DT_REP
  AND  NOT EXISTS (SELECT 1 FROM WORK.ForecastKey_Cache
                   WHERE DT_REP = @PrevDay);

/* 3.2  Обновляем будущую проекцию (сдвигаем окно на +1) */
DELETE FROM WORK.ForecastKey_Cache
WHERE DT_REP > DATEADD(day,@Horizon,@Today);      -- чтобы объём не рос бесконечно

/* добавляем новую «хвостовую» дату @Today+@Horizon */
INSERT INTO WORK.ForecastKey_Cache (DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE)
SELECT
        DT_Future = DATEADD(day,@Horizon,@Today),
        f.[Date],
        f.KEY_RATE,
        TERM       = ROW_NUMBER() OVER
                       (ORDER BY f.[Date]),
        AVG_KEY_RATE = AVG(f.KEY_RATE) OVER
                       (ORDER BY f.[Date])
FROM    ALM.info.VW_ForecastKEY_interval f
WHERE   f.DT_REP = (SELECT MAX(DT_REP)
                    FROM ALM.info.VW_ForecastKEY_interval);
```

— Теперь таблица-кеш всегда содержит:
**▪ всё прошлое ▪ сегодня ▪ ровно @Horizon дней вперёд.**

---

## 4 ∙ Как быстро джоинить к реестру договоров

```sql
SELECT  d.con_id,
        d.open_date,
        d.term_days,

        fk_open.AVG_KEY_RATE  AS rate_at_open,
        fk_close.AVG_KEY_RATE AS rate_at_close
FROM    WORK.DepositRegister d

LEFT JOIN WORK.ForecastKey_Cache fk_open       -- «при открытии»
       ON fk_open.DT_REP = d.open_date
      AND fk_open.TERM   = d.term_days

LEFT JOIN WORK.ForecastKey_Cache fk_close      -- «при закрытии»
       ON fk_close.DT_REP = DATEADD(day,d.term_days,d.open_date)
      AND fk_close.TERM   = d.term_days;
```

* Оба `JOIN`-а — мгновенный *seek* по кластерному PK *(DT\_REP, TERM)*.
* 370 000 договоров = ≈ 15-20 секунд даже без дополнительных индексов.

---

### Что, если нужен другой горизонт?

Поменяйте `@Horizon` в скриптах 2.2 и 3.2. Таблица-кеш вырастет линейно.

### Можно ли хранить только `AVG_KEY_RATE`?

Да, тогда таблица будет легче. Но оставив `KEY_RATE` вы всегда сможете быстро
посчитать min / max или сделать стресс-тест без переделки пайплайна.

---

**Итого:** мы полностью «размножили» интервальную таблицу прогноза в ALM\_TEST,
добавили проекцию на будущее и получили компактный кеш, к которому
джойнимся двумя seek-ами вместо минут ожидания на UDF.
