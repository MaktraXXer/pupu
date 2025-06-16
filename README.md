```sql
/*======================================================================
  1.  ТАБЛИЦЫ
======================================================================*/
IF OBJECT_ID('dbo.Liq_Outflow','U') IS NOT NULL DROP TABLE dbo.Liq_Outflow;
IF OBJECT_ID('dbo.Liq_Balance','U') IS NOT NULL DROP TABLE dbo.Liq_Balance;
GO

/* ---------- Балансы (ФЛ / ЮЛ) ------------------------------------- */
CREATE TABLE dbo.Liq_Balance (
    dt_rep       date           NOT NULL,
    addend_name  nvarchar(50)   NOT NULL,      -- 'ЮЛ' | 'Средства ФЛ'
    balance_rub  decimal(38,2)  NOT NULL,
    CONSTRAINT PK_Liq_Balance PRIMARY KEY CLUSTERED (dt_rep, addend_name)
);
GO

/* ---------- Оттоки по нормативам ---------------------------------- */
CREATE TABLE dbo.Liq_Outflow (
    dt_rep       date           NOT NULL,
    addend_name  nvarchar(50)   NOT NULL,
    normativ     nvarchar(50)   NOT NULL,      -- напр. 'ГВ 70'
    amount_sum   decimal(38,2)  NOT NULL,
    CONSTRAINT PK_Liq_Outflow PRIMARY KEY CLUSTERED (dt_rep, addend_name, normativ),
    CONSTRAINT FK_Outflow_Bal FOREIGN KEY (dt_rep, addend_name)
        REFERENCES dbo.Liq_Balance (dt_rep, addend_name)
        ON DELETE CASCADE
);
GO

/* индекс по дате для быстрых выборок */
CREATE NONCLUSTERED INDEX IX_Liq_Balance_dt ON dbo.Liq_Balance(dt_rep);
GO


/*======================================================================
  2.  ПРОЦЕДУРА – СОХРАНЕНИЕ БАЛАНСОВ
======================================================================*/
CREATE OR ALTER PROCEDURE dbo.usp_SaveBalance
    @dt_rep date                 -- дата, за которую фиксируем срез
AS
BEGIN
    SET NOCOUNT ON;

    /* ---------- собираем балансы ЮЛ и ФЛ на @dt_rep -------------- */
    WITH balances AS (
        /* ===== ЮЛ ===== */
        SELECT
            @dt_rep                        AS dt_rep,
            N'ЮЛ'                          AS addend_name,
            SUM(b.BALANCE_RUB)             AS balance_rub
        FROM   ALM_TEST.[WORK].[GroupDepositInterestsRate_UL_matur_pdr_fo] b WITH (NOLOCK)
        WHERE  [TYPE]            = N'Срез'
          AND  [CLI_SUBTYPE]     = N'ЮЛ'
          AND  [TERM_GROUP]      = N'all termgroup'
          AND  [SEG_NAME]        = N'all segment'
          AND  [IS_PDR]          = N'all PDR'
          AND  [IS_FINANCE_LCR]  = N'all FINANCE_LCR'
          AND  [IS_OPTION]       = N'all option_type'
          AND  [MARGIN_TYPE]     = N'all margin'
          AND  CAST([Date] AS date) = @dt_rep
        GROUP BY
            CAST([Date] AS date)

        UNION ALL

        /* ===== ФЛ ===== */
        SELECT
            @dt_rep                        AS dt_rep,
            N'Средства ФЛ'                 AS addend_name,
            SUM(ISNULL(sOUT_RUB,0)) * 1000 AS balance_rub
        FROM   ALM.ALM.VW_alm_balance_AGG_ALMREPORT WITH (NOLOCK)
        WHERE  CAST(dt_rep AS date) = @dt_rep
          AND  AP               = N'Пассив'
          AND  section_name IN  (N'До востребования', N'Срочные ', N'Накопительный счёт')
          AND  block_name       = N'Привлечение ФЛ'
          AND  TSegmentName IN  (N'ДЧБО', N'Розничный Бизнес')
        GROUP BY
            CAST(dt_rep AS date)
    )

    /* ---------- вставка / обновление ----------------------------- */
    MERGE dbo.Liq_Balance AS tgt
    USING balances        AS src
      ON  tgt.dt_rep      = src.dt_rep
      AND tgt.addend_name = src.addend_name
    WHEN MATCHED THEN
         UPDATE SET balance_rub = src.balance_rub
    WHEN NOT MATCHED THEN
         INSERT (dt_rep, addend_name, balance_rub)
         VALUES (src.dt_rep, src.addend_name, src.balance_rub);
END
GO


/*======================================================================
  3.  ПРОЦЕДУРА – СОХРАНЕНИЕ ОТТОКОВ
======================================================================*/
CREATE OR ALTER PROCEDURE dbo.usp_SaveOutflow
    @dt_rep       date,
    @addend_name  nvarchar(50),   -- 'ЮЛ' | 'Средства ФЛ'
    @normativ     nvarchar(50),   -- напр. 'ГВ 70'
    @amount_sum   decimal(38,2)
AS
BEGIN
    SET NOCOUNT ON;

    /* баланс должен быть занесён раньше */
    IF NOT EXISTS (SELECT 1
                   FROM   dbo.Liq_Balance
                   WHERE  dt_rep      = @dt_rep
                     AND  addend_name = @addend_name)
    BEGIN
        THROW 51000, N'Сначала сохраните баланс (usp_SaveBalance).', 1;
    END

    MERGE dbo.Liq_Outflow AS tgt
    USING (SELECT @dt_rep      AS dt_rep,
                  @addend_name AS addend_name,
                  @normativ    AS normativ,
                  @amount_sum  AS amount_sum) AS src
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


/*======================================================================
  4.  ВИТРИНА – ОТНОШЕНИЕ ОТТОК / БАЛАНС
======================================================================*/
CREATE OR ALTER VIEW dbo.vw_Liq_Ratio
AS
SELECT
    b.dt_rep,
    b.addend_name,
    piv.[ГВ 70]      AS Ratio_GV70,
    piv.[ГВ 100]     AS Ratio_GV100,
    piv.[АЛМ stress] AS Ratio_ALM_Stress
FROM dbo.Liq_Balance AS b
OUTER APPLY (
    SELECT *
    FROM (
        SELECT
            o.normativ,
            CAST( o.amount_sum /
                  NULLIF(b.balance_rub,0) AS decimal(28,8) ) AS ratio
        FROM dbo.Liq_Outflow o
        WHERE o.dt_rep      = b.dt_rep
          AND o.addend_name = b.addend_name
    ) AS s
    PIVOT (
        MAX(ratio) FOR normativ IN (
            [ГВ 70],
            [ГВ 100],
            [АЛМ stress]
        )
    ) AS p
) AS piv;
GO
```
