### Да, лучше “привязаться” не к `GETDATE()`, а к последней фактически загруженной дате прогноза (`@LastRep`)

Так мы гарантированно не попытаемся включить в кэш не-существующий `DT_REP = today`, а вся логика «история + 90 дней вперёд» остаётся корректной.

Ниже оба скрипта целиком — уже с поправкой.
Ключевая идея:

```sql
DECLARE @Anchor DATE = (SELECT MAX(DT_REP) FROM ALM.info.VW_ForecastKEY_interval);
```

`@Anchor` — это **t-1** от `GETDATE()` (или более ранний, если выгрузка задержалась).
История копируется до `@Anchor`, а проекция строится от `@Anchor+1` до `@Anchor+@Horizon`.

---

## 1 · Инициализация (разовая)

```sql
USE ALM_TEST;
GO
IF OBJECT_ID('WORK.ForecastKey_Cache','U') IS NOT NULL
    DROP TABLE WORK.ForecastKey_Cache;
GO
CREATE TABLE WORK.ForecastKey_Cache
( DT_REP       DATE         NOT NULL,
  [Date]       DATE         NOT NULL,
  KEY_RATE     DECIMAL(9,4) NOT NULL,
  TERM         INT          NOT NULL,
  AVG_KEY_RATE DECIMAL(9,4) NOT NULL,
  CONSTRAINT PK_FKCache PRIMARY KEY CLUSTERED (DT_REP, TERM)
);
GO

DECLARE @Horizon INT  = 90;
DECLARE @Anchor  DATE = (SELECT MAX(DT_REP)
                         FROM ALM.info.VW_ForecastKEY_interval); -- t-1

/* ---------- 1.1  История до @Anchor ---------- */
INSERT INTO WORK.ForecastKey_Cache (DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE)
SELECT cal.[Date], fkey.[Date], fkey.KEY_RATE,
       ROW_NUMBER() OVER (PARTITION BY cal.[Date] ORDER BY fkey.[Date]),
       AVG(fkey.KEY_RATE) OVER (PARTITION BY cal.[Date] ORDER BY fkey.[Date])
FROM   ALM.info.VW_ForecastKEY_interval fkey  WITH (NOLOCK)
JOIN   ALM.info.VW_calendar          cal   WITH (NOLOCK)
  ON   cal.[Date] >= fkey.DT_REP
 AND   cal.[Date] <  fkey.DT_REP_NEXT
WHERE  cal.[Date] <= @Anchor;

/* ---------- 1.2  Проекция вперёд: 1 … @Horizon ---------- */
;WITH nums AS (
      SELECT TOP (@Horizon) ROW_NUMBER() OVER (ORDER BY (SELECT 0)) AS n
      FROM sys.all_objects )
INSERT INTO WORK.ForecastKey_Cache (DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE)
SELECT DATEADD(day,n,@Anchor)                AS DT_REP,
       f.[Date],
       f.KEY_RATE,
       ROW_NUMBER() OVER
           (PARTITION BY DATEADD(day,n,@Anchor) ORDER BY f.[Date])   AS TERM,
       AVG(f.KEY_RATE) OVER
           (PARTITION BY DATEADD(day,n,@Anchor) ORDER BY f.[Date])   AS AVG_KEY_RATE
FROM   nums
JOIN   ALM.info.VW_ForecastKEY_interval f  WITH (NOLOCK)
  ON   f.DT_REP = @Anchor                  -- прогноз последнего снимка
 AND   f.[Date] >= DATEADD(day,n,@Anchor)   -- отрезаем прошедшие дни
OPTION (MAXRECURSION 0);
```

---

## 2 · Ежедневный инкремент (для SQL Agent)

```sql
DECLARE @Horizon INT  = 90;
DECLARE @Anchor  DATE = (SELECT MAX(DT_REP)
                         FROM ALM.info.VW_ForecastKEY_interval); -- t-1
DECLARE @PrevDay DATE = DATEADD(day,-1,@Anchor);                 -- t-2
DECLARE @NewTail DATE = DATEADD(day,@Horizon,@Anchor);           -- t-1 + 90

/* 2.1  Подливаем вчерашний факт, если его ещё нет */
INSERT INTO WORK.ForecastKey_Cache (DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE)
SELECT cal.[Date], fkey.[Date], fkey.KEY_RATE,
       ROW_NUMBER() OVER (PARTITION BY cal.[Date] ORDER BY fkey.[Date]),
       AVG(fkey.KEY_RATE) OVER (PARTITION BY cal.[Date] ORDER BY fkey.[Date])
FROM   ALM.info.VW_ForecastKEY_interval fkey
JOIN   ALM.info.VW_calendar cal
  ON   cal.[Date] >= fkey.DT_REP
 AND   cal.[Date] <  fkey.DT_REP_NEXT
WHERE  cal.[Date] = @PrevDay
  AND  NOT EXISTS (SELECT 1
                   FROM WORK.ForecastKey_Cache
                   WHERE DT_REP = @PrevDay);

/* 2.2  Удаляем возможный дубль + вставляем новый хвост */
DELETE FROM WORK.ForecastKey_Cache
WHERE  DT_REP = @NewTail;

INSERT INTO WORK.ForecastKey_Cache (DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE)
SELECT @NewTail, f.[Date], f.KEY_RATE,
       ROW_NUMBER() OVER (ORDER BY f.[Date])                 AS TERM,
       AVG(f.KEY_RATE) OVER (ORDER BY f.[Date])              AS AVG_KEY_RATE
FROM   ALM.info.VW_ForecastKEY_interval f
WHERE  f.DT_REP = @Anchor          -- тот же самый последний снимок
  AND  f.[Date] >= @NewTail;       -- только будущие даты
```

---

### Проверка

```sql
/* убедимся, что есть ровно 90 будущих DT_REP */
SELECT COUNT(DISTINCT DT_REP) AS future_days
FROM   WORK.ForecastKey_Cache
WHERE  DT_REP > @Anchor;         -- должно быть = @Horizon
```

---

### Что изменилось и почему теперь корректно

| Было                                            | Стало                                                                       |
| ----------------------------------------------- | --------------------------------------------------------------------------- |
| `@Today = GETDATE()`                            | `@Anchor = MAX(DT_REP)` — последняя реально загруженная дата.               |
| Первичное заполнение включало `@Today+@Horizon` | Теперь проекция идёт 1 … `@Horizon`, а **новый хвост** добавляет агент-шаг. |
| `DELETE` отсекает “> Today + Horizon”           | Теперь отсекаем **только** новый хвост перед вставкой, дубликатов нет.      |

Так кэш всегда содержит:

* все даты до `@Anchor` (исторические),
* **ровно** `@Horizon` будущих дат (`@Anchor+1 … @Anchor+@Horizon`).

И ни одного PK-конфликта.
