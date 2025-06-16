/*======================================================================
  liq.usp_SaveOutflow   –  загрузка оттоков за диапазон дат
======================================================================*/
CREATE OR ALTER PROCEDURE liq.usp_SaveOutflow
       @date_from date,          -- начало (включительно)
       @date_to   date,          -- конец  (включительно)
       @normativ  nvarchar(50)   -- 'ГВ 70' | 'ГВ 100' | 'АЛМ stress' …
AS
BEGIN
    SET NOCOUNT ON;

    /* --- валидация параметров ------------------------------------ */
    IF @date_from IS NULL OR @date_to IS NULL
    BEGIN
        THROW 51000, N'Обе даты обязательны.', 1;
    END;

    IF @date_from > @date_to
    BEGIN
        THROW 51000, N'@date_from не может быть позже @date_to.', 1;
    END;

    /* --- 1. собираем оттоки -------------------------------------- */
    ;WITH base AS (
        SELECT
            CAST(DT_REP AS date) AS dt_rep,
            CASE WHEN ADDEND_NAME = N'Средства ФЛ'
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END   AS addend_name,
            SUM(AMOUNT_RUB_MOD)    AS amount_sum
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
          AND DT_REP <  DATEADD(DAY,1,@date_to)      -- ≤ @date_to
        GROUP BY
            CAST(DT_REP AS date),
            CASE WHEN ADDEND_NAME = N'Средства ФЛ'
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END
    )

    /* --- 2. проверяем, что баланс есть на все даты ---------------- */
    IF EXISTS (
        SELECT 1
        FROM   base b
        LEFT   JOIN liq.Liq_Balance lb
               ON  lb.dt_rep      = b.dt_rep
               AND lb.addend_name = b.addend_name
        WHERE  lb.dt_rep IS NULL
    )
    BEGIN
        THROW 51000,
      N'Для части дат/сегментов нет балансов. Сначала вызовите liq.usp_SaveBalance.', 1;
    END;

    /* --- 3. MERGE в liq.Liq_Outflow ------------------------------- */
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


/* =====================  Пример запуска  ===================== */
DECLARE @from date = '2025-05-01',
        @to   date = '2025-05-31';

/* balances уже залиты через liq.usp_SaveBalance_Range */

EXEC liq.usp_SaveOutflow @date_from=@from, @date_to=@to, @normativ=N'ГВ 70';
EXEC liq.usp_SaveOutflow @date_from=@from, @date_to=@to, @normativ=N'ГВ 100';
EXEC liq.usp_SaveOutflow @date_from=@from, @date_to=@to, @normativ=N'АЛМ stress';

SELECT * 
FROM   liq.vw_Liq_Ratio
WHERE  dt_rep BETWEEN @from AND @to
ORDER  BY dt_rep, addend_name;
