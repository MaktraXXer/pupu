### Проще всего — «погонять» существующую **liq.usp\_SaveBalance** в цикле

(процедура уже умеет сама MERGE-ить, поэтому повторные даты просто обновятся).

```sql
/*--------------------------------------------------------------------
  Записываем балансы за ВЕСЬ апрель-2025
--------------------------------------------------------------------*/
DECLARE 
    @d       date  = '2025-04-01',          -- начало месяца
    @finish  date  = DATEADD(DAY,-1,DATEADD(MONTH,1,@d));  -- 2025-04-30

WHILE @d <= @finish
BEGIN
    EXEC liq.usp_SaveBalance @dt_rep = @d;  -- фиксируем день
    SET @d = DATEADD(DAY, 1, @d);           -- следующий день
END
```

> ⚠️ Если нужно другой месяц — поменяйте начальное значение `@d`;
> расчёт `@finish` оставьте как есть (он «подставит» последний день месяца автоматически).

---

## Нужен такой вызов регулярно?

Сделайте маленькую «обёртку»-процедуру для диапазона дат:

```sql
CREATE OR ALTER PROCEDURE liq.usp_SaveBalance_Range
      @date_from date,
      @date_to   date
AS
BEGIN
    SET NOCOUNT ON;

    IF @date_from > @date_to
       THROW 51000, N'@date_from не может быть позже @date_to', 1;

    DECLARE @d date = @date_from;

    WHILE @d <= @date_to
    BEGIN
        EXEC liq.usp_SaveBalance @dt_rep = @d;
        SET @d = DATEADD(DAY,1,@d);
    END
END;
GO
```

**Пример вызова на май-2025:**

```sql
EXEC liq.usp_SaveBalance_Range 
     @date_from = '2025-05-01',
     @date_to   = '2025-05-31';
```

Обе схемы работают с тем же MERGE-логическим идентификатором
`(dt_rep, addend_name)`, поэтому ничего лишнего не продублируется.
