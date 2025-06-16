CREATE OR ALTER PROCEDURE liq.usp_SaveOutflow
    @dt_rep    date,          -- дата среза
    @normativ  nvarchar(50)   -- 'ГВ 70', 'ГВ 100', 'АЛМ stress' …
AS
BEGIN
    SET NOCOUNT ON;

    /* ---------- 1.  оттоки из источника --------------------------- */
    ;WITH base AS (
        SELECT
            CAST(DT_REP AS date) AS dt_rep,
            CASE WHEN ADDEND_NAME = N'Средства ФЛ'
                 THEN N'Средства ФЛ'
                 ELSE N'ЮЛ' END  AS addend_name,
            SUM(AMOUNT_RUB_MOD)  AS amount_sum
        FROM LIQUIDITY.ratio.VW_SH_Ratio_Agg_LVL2_Fact WITH (NOLOCK)
        WHERE OrganizationName = N'Банк ДОМ.РФ'
          AND ADDEND_TYPE      = N'Оттоки'
          AND ADDEND_DESCR    <> N'Аккредитивы'
          AND ADDEND_NAME IN (
                 N'Средства ФЛ',
                 N'Средства ЮЛ',
                 N'Средства Ф/О в рамках станд. продуктов',
                 N'Средства Ф/О'
              )
          AND CAST(DT_REP AS date) = @dt_rep
        GROUP BY
            CAST(DT_REP AS date),
            CASE WHEN ADDEND_NAME = N'Средства ФЛ'
                 THEN N'Средства ФЛ'
                 ELSE N'ЮЛ' END
    )

    /* ---------- 2.  проверяем наличие балансов -------------------- */
    IF EXISTS (
        SELECT 1
        FROM   base b
        WHERE NOT EXISTS (
            SELECT 1
            FROM   liq.Liq_Balance lb
            WHERE  lb.dt_rep      = b.dt_rep
              AND  lb.addend_name = b.addend_name
        )
    )
    BEGIN
        THROW 51000,
              N'Сначала зафиксируйте балансы через liq.usp_SaveBalance.',
              1;
    END

    /* ---------- 3.  MERGE в liq.Liq_Outflow ----------------------- */
    MERGE liq.Liq_Outflow AS tgt
    USING (
        SELECT
            dt_rep,
            addend_name,
            @normativ  AS normativ,
            amount_sum
        FROM base
    ) AS src
      ON  tgt.dt_rep      = src.dt_rep
     AND  tgt.addend_name = src.addend_name
     AND  tgt.normativ    = src.normativ
    WHEN MATCHED THEN
         UPDATE SET amount_sum = src.amount_sum
    WHEN NOT MATCHED THEN
         INSERT (dt_rep, addend_name, normativ, amount_sum)
         VALUES (src.dt_rep, src.addend_name, src.normativ, src.amount_sum);
END
GO
