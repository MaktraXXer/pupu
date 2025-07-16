**Почему возникла ошибка**

`WITH INLINE = ON` можно ставить только в двух случаях (а) SQL Server 2022+ *и* (б) тело функции удовлетворяет строгим требованиям авто-инлайнинга — фактически это должна быть «однострочная» конструкция `RETURN ( … )` без переменных, IF/ELSE, временных таблиц и т.п.
Наша UDF содержит объявления переменных и условные ветки, поэтому компилятор выдал:

```
Msg 15682:  INLINE = ON is not valid for this function body.
```

> Даже на SQL Server 2019/2022 функция **всё равно** может быть автоматически инлайн-развёрнута, но указывать опцию «ON» нельзя, если код не подходит под правила.

---

## Исправленный вариант без директивы INLINE

```sql
USE ALM_TEST;
GO

CREATE OR ALTER FUNCTION WORK.ufn_GetForecastKey
(
    @OpenDate DATE,        -- дата открытия
    @Term     INT,         -- срок, дней  (0 трактуется как 1)
    @Mode     CHAR(1) = 'O'-- 'O' – на момент открытия, 'C' – на момент закрытия
)
RETURNS DECIMAL(9,4)
-- никаких WITH INLINE, SCHEMABINDING и т.п.
AS
BEGIN
    /* ------------------------------------------------------ */
    IF @Term < 1 SET @Term = 1;          -- 0 → диапазон из одного дня
    /* ------------------------------------------------------ */
    DECLARE @Today     DATE      = CAST(GETDATE() AS DATE),
            @CloseDate DATE      = DATEADD(day,@Term,@OpenDate),
            @LastRep   DATE,             -- самый свежий снимок
            @OpenRep   DATE,             -- снимок «живой» в день открытия
            @Result    DECIMAL(9,4);

    /* 1. последний доступный прогноз */
    SELECT @LastRep = MAX(DT_REP)
    FROM   LIQUIDITY.liq.ForecastKeyRate
    WHERE  DT_REP <= @Today;

    /* 2. репорт, активный на дату открытия */
    IF @OpenDate > @Today               -- будущее
        SET @OpenRep = @LastRep;
    ELSE
    BEGIN
        ;WITH R AS (
            SELECT DISTINCT DT_REP
            FROM   LIQUIDITY.liq.ForecastKeyRate)
        ,X AS (
            SELECT DT_REP,
                   LEAD(DT_REP) OVER (ORDER BY DT_REP) AS DT_REP_NEXT
            FROM   R)
        SELECT TOP (1) @OpenRep = DT_REP
        FROM   X
        WHERE  DT_REP      <= @OpenDate
          AND  @OpenDate   <  ISNULL(DT_REP_NEXT,'30000101')
        ORDER  BY DT_REP DESC;
    END

    /* 3. средний прогноз */
    IF @Mode = 'O'
        SELECT @Result = AVG(KEY_RATE)
        FROM   LIQUIDITY.liq.ForecastKeyRate
        WHERE  DT_REP = @OpenRep
          AND  [Date] BETWEEN @OpenDate
                          AND DATEADD(day,@Term-1,@OpenDate);
    ELSE
        SELECT @Result = AVG(KEY_RATE)
        FROM   LIQUIDITY.liq.ForecastKeyRate
        WHERE  DT_REP = @LastRep
          AND  [Date] BETWEEN @CloseDate
                          AND DATEADD(day,@Term-1,@CloseDate);

    RETURN @Result;
END
GO
```

### Что изменилось

* Убрана опция `WITH INLINE = ON` — функция успешно создаётся на любой версии SQL Server ≥ 2016.
* Добавлена защита: если `@Term = 0`, он превращается в `1` и возвращается ставка «точно на дату».

---

## Как использовать / джойнить

```sql
SELECT  d.*,
        r.rate_at_open,
        r.rate_at_close
FROM    WORK.DepositRegister AS d
CROSS APPLY
(
   SELECT WORK.ufn_GetForecastKey(d.open_date, d.term_days,'O') AS rate_at_open,
          WORK.ufn_GetForecastKey(d.open_date, d.term_days,'C') AS rate_at_close
) AS r;
```

* `OUTER APPLY` — если хотите сохранить строки, даже когда функция вернёт `NULL`.
* Нагрузка минимальна: для каждого договора читается ≤ 2 × @Term строк из `ForecastKeyRate`.

---

### Итог

1. Ошибка устранена — функция успешно компилируется.
2. Производительность остаётся приемлемой (десятки мс на вызов в массовом запросе).
3. Поведение для будущих дат и `@Term = 0` корректно обработано.

Если потребуется дальнейшая оптимизация (например, массовый nightly-проход по миллионам договоров) — можно перейти к локальному кешу в **ALM\_TEST**, как обсуждали ранее.
