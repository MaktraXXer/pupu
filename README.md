Ниже ‒ полная пере-версия **liq.usp\_SaveOutflow**,
которая принимает диапазон дат `@date_from … @date_to` и один норматив.
Процедура берёт все нужные оттоки за указанный интервал, проверяет, что
для каждой даты уже зафиксирован баланс (через `liq.usp_SaveBalance`),
и одной `MERGE`-операцией добавляет/обновляет строки в `liq.Liq_Outflow`.

```sql
/*======================================================================
  liq.usp_SaveOutflow_Range     ──  оттоки за диапазон дат
======================================================================*/
CREATE OR ALTER PROCEDURE liq.usp_SaveOutflow
       @date_from date,          -- начиная с этой даты  (включительно)
       @date_to   date,          -- по эту дату          (включительно)
       @normativ  nvarchar(50)   -- 'ГВ 70' | 'ГВ 100' | 'АЛМ stress' …
AS
BEGIN
    SET NOCOUNT ON;

    /* ---- вменяемость диапазона ----------------------------------- */
    IF @date_from IS NULL  OR  @date_to IS NULL
        THROW 51000, N'Обе даты обязательны', 1;

    IF @date_from > @date_to
        THROW 51000, N'@date_from не может быть позже @date_to', 1;

    /* ---- 1.  собираем оттоки за интервал -------------------------- */
    ;WITH base AS (
        SELECT
            CAST(DT_REP AS date)                                     AS dt_rep,
            CASE WHEN ADDEND_NAME = N'Средства ФЛ'
                 THEN N'Средства ФЛ'
                 ELSE N'ЮЛ' END                                      AS addend_name,
            SUM(AMOUNT_RUB_MOD)                                      AS amount_sum
        FROM LIQUIDITY.ratio.VW_SH_Ratio_Agg_LVL2_Fact  WITH (NOLOCK)
        WHERE OrganizationName = N'Банк ДОМ.РФ'
          AND ADDEND_TYPE      = N'Оттоки'
          AND ADDEND_DESCR    <> N'Аккредитивы'
          AND ADDEND_NAME IN (
                 N'Средства ФЛ',
                 N'Средства ЮЛ',
                 N'Средства Ф/О в рамках станд. продуктов',
                 N'Средства Ф/О'
              )
          AND DT_REP >= @date_from
          AND DT_REP <  DATEADD(DAY, 1, @date_to)       -- «≤ @date_to»
        GROUP BY
            CAST(DT_REP AS date),
            CASE WHEN ADDEND_NAME = N'Средства ФЛ'
                 THEN N'Средства ФЛ'
                 ELSE N'ЮЛ' END
    )

    /* ---- 2.  проверяем, что балансы есть для ВСЕХ дат ------------- */
    IF EXISTS (
        SELECT 1
        FROM   base b
        LEFT  JOIN liq.Liq_Balance lb
               ON  lb.dt_rep      = b.dt_rep
               AND lb.addend_name = b.addend_name
        WHERE  lb.dt_rep IS NULL                 -- чего-то не хватает
    )
    BEGIN
        THROW 51000,
              N'Для части дат/сегментов нет балансов. '
              + N'Сначала запустите liq.usp_SaveBalance для нужного диапазона.',
              1;
    END

    /* ---- 3.  MERGE сразу по всему набору -------------------------- */
    MERGE liq.Liq_Outflow AS tgt
    USING (
        SELECT
            dt_rep,
            addend_name,
            @normativ AS normativ,
            amount_sum
        FROM base
    ) AS src
      ON  tgt.dt_rep      = src.dt_rep
      AND tgt.addend_name = src.addend_name
      AND tgt.normativ    = src.normativ
    WHEN MATCHED THEN
         UPDATE SET amount_sum = src.amount_sum
    WHEN NOT MATCHED THEN
         INSERT (dt_rep, addend_name, normativ, amount_sum)
         VALUES (src.dt_rep, src.addend_name, src.normativ, src.amount_sum);
END
GO
```

---

### Как вызывать

```sql
/* фиксируем балансы за месяц (если ещё не сделали) */
EXEC liq.usp_SaveBalance_Range 
     @date_from = '2025-05-01',
     @date_to   = '2025-05-31';

/* пишем оттоки по трём нормативам тем же диапазоном */
EXEC liq.usp_SaveOutflow @date_from='2025-05-01', @date_to='2025-05-31', @normativ=N'ГВ 70';
EXEC liq.usp_SaveOutflow @date_from='2025-05-01', @date_to='2025-05-31', @normativ=N'ГВ 100';
EXEC liq.usp_SaveOutflow @date_from='2025-05-01', @date_to='2025-05-31', @normativ=N'АЛМ stress';

/* проверяем результат */
SELECT * 
FROM   liq.vw_Liq_Ratio
WHERE  dt_rep BETWEEN '2025-05-01' AND '2025-05-31'
ORDER  BY dt_rep, addend_name;
```

Так одним вызовом вы загружаете оттоки сразу за весь нужный период, не бегая по датам в цикле.
