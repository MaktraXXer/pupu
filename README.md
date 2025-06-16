Вот просто-работающая версия `liq.usp_SaveOutflow` для диапазона дат
— без всяких `IF … EXISTS`, без выброса ошибок.
Если в исходнике на какую-то дату нет данных по сегменту, в таблицу уйдёт строка с `amount_sum = NULL`.

```sql
/*======================================================================
  liq.usp_SaveOutflow   –  грузим оттоки за диапазон дат
       • date_from / date_to  –  включительно
       • normativ             –  'ГВ 70', 'ГВ 100', 'АЛМ stress', …
======================================================================*/
CREATE OR ALTER PROCEDURE liq.usp_SaveOutflow
       @date_from date,
       @date_to   date,
       @normativ  nvarchar(50)
AS
BEGIN
    SET NOCOUNT ON;

    /* ---------- календарь дат ------------------------------------ */
    ;WITH cte_dates AS (
        SELECT @date_from AS dt
        UNION ALL
        SELECT DATEADD(DAY,1,dt)
        FROM   cte_dates
        WHERE  dt < @date_to
    ),
    segments AS (                -- два нужных сегмента
        SELECT N'ЮЛ' AS addend_name
        UNION ALL
        SELECT N'Средства ФЛ'
    ),

    /* ---------- оттоки из источника ------------------------------ */
    base AS (
        SELECT
            CAST(DT_REP AS date)                                    AS dt_rep,
            CASE WHEN ADDEND_NAME = N'Средства ФЛ'
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END                 AS addend_name,
            SUM(AMOUNT_RUB_MOD)                                     AS amount_sum
        FROM LIQUIDITY.ratio.VW_SH_Ratio_Agg_LVL2_Fact  WITH (NOLOCK)
        WHERE OrganizationName = N'Банк ДОМ.РФ'
          AND ADDEND_TYPE      = N'Оттоки'
          AND ADDEND_DESCR    <> N'Аккредитивы'
          AND ADDEND_NAME IN (N'Средства ФЛ',N'Средства ЮЛ',
                              N'Средства Ф/О в рамках станд. продуктов',
                              N'Средства Ф/О')
          AND DT_REP >= @date_from
          AND DT_REP <  DATEADD(DAY,1,@date_to)           -- «≤ @date_to»
        GROUP BY
            CAST(DT_REP AS date),
            CASE WHEN ADDEND_NAME = N'Средства ФЛ'
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END
    ),

    /* ---------- «решётка» всех дат × сегментов ------------------- */
    grid AS (
        SELECT d.dt       AS dt_rep,
               s.addend_name
        FROM   cte_dates  AS d
        CROSS  JOIN segments AS s
    ),

    /* ---------- финальный набор для MERGE ------------------------ */
    src AS (
        SELECT
            g.dt_rep,
            g.addend_name,
            @normativ                AS normativ,
            b.amount_sum             -- NULL если в base нет строки
        FROM   grid g
        LEFT   JOIN base b
               ON  b.dt_rep      = g.dt_rep
               AND b.addend_name = g.addend_name
    )

    /* ---------- MERGE в liq.Liq_Outflow -------------------------- */
    MERGE liq.Liq_Outflow AS tgt
    USING src
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
OPTION (MAXRECURSION 32767);   -- чтобы рекурсивный CTE спокойно брал месяц-два
```

### Пример вызова

```sql
DECLARE @from date = '2025-05-01',
        @to   date = '2025-05-31';

/* balances уже залиты через liq.usp_SaveBalance_Range */

EXEC liq.usp_SaveOutflow @date_from=@from, @date_to=@to, @normativ=N'ГВ 70';
EXEC liq.usp_SaveOutflow @date_from=@from, @date_to=@to, @normativ=N'ГВ 100';
EXEC liq.usp_SaveOutflow @date_from=@from, @date_to=@to, @normativ=N'АЛМ stress';
```

* Для каждой даты в интервале и для обоих сегментов всегда будет строка в `liq.Liq_Outflow`.
* Если исходный отчёт оттоков ничего не выдал — `amount_sum` останется `NULL`, так что витрина покажет `NULL`-долю.
