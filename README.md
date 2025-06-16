```sql
/*======================================================================
  0.  КОНТЕКСТ И СХЕМА
======================================================================*/
USE ALM_TEST;
GO

/*  Отдельная схема для всего, что ниже */
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = 'liq')
    EXEC ('CREATE SCHEMA liq AUTHORIZATION dbo');
GO


/*======================================================================
  1.  ТАБЛИЦЫ
======================================================================*/
IF OBJECT_ID('liq.Liq_Outflow','U') IS NOT NULL DROP TABLE liq.Liq_Outflow;
IF OBJECT_ID('liq.Liq_Balance','U') IS NOT NULL DROP TABLE liq.Liq_Balance;
GO

/* ---------- Балансы (ФЛ / ЮЛ) ------------------------------------- */
CREATE TABLE liq.Liq_Balance (
    dt_rep       date           NOT NULL,
    addend_name  nvarchar(50)   NOT NULL,      -- 'ЮЛ' | 'Средства ФЛ'
    balance_rub  decimal(38,2)  NOT NULL,
    CONSTRAINT PK_Liq_Balance PRIMARY KEY CLUSTERED (dt_rep, addend_name)
);
GO

/* ---------- Оттоки по нормативам ---------------------------------- */
CREATE TABLE liq.Liq_Outflow (
    dt_rep       date           NOT NULL,
    addend_name  nvarchar(50)   NOT NULL,
    normativ     nvarchar(50)   NOT NULL,      -- напр. 'ГВ 70'
    amount_sum   decimal(38,2)  NOT NULL,
    CONSTRAINT PK_Liq_Outflow PRIMARY KEY CLUSTERED (dt_rep, addend_name, normativ),
    CONSTRAINT FK_Outflow_Bal FOREIGN KEY (dt_rep, addend_name)
        REFERENCES liq.Liq_Balance (dt_rep, addend_name)
        ON DELETE CASCADE
);
GO

/* индекс по дате для быстрых выборок */
CREATE NONCLUSTERED INDEX IX_Liq_Balance_dt
    ON liq.Liq_Balance (dt_rep);
GO


/*======================================================================
  2.  ПРОЦЕДУРА – СОХРАНЕНИЕ БАЛАНСОВ
======================================================================*/
CREATE OR ALTER PROCEDURE liq.usp_SaveBalance
    @dt_rep date                 -- дата, за которую фиксируем срез
AS
BEGIN
    SET NOCOUNT ON;

    /* ---------- балансы ЮЛ и ФЛ на @dt_rep ------------------------ */
    WITH ul_balance AS (
        /* ——— ЮЛ через VW_balance_rest_all (фильтры «Старый ГВ») ——— */
        WITH base AS (
            SELECT
                b.DT_REP,
                b.CON_ID,
                ABS(b.OUT_RUB)               AS OUT_RUB,
                LOWER(b.ACC_ROLE)            AS acc_role,
                cli.IS_FINANCE_LCR           AS is_finance,
                cli.IS_SSV                   AS is_ssv,
                cli.ISDOMRF                  AS is_domrf,
                CASE WHEN b.PROD_ID IN (398,399,400) THEN 1 ELSE 0 END AS is_broker,
                CASE WHEN LOWER(b.CON_TYPE) = 'loc' THEN 1 ELSE 0 END  AS is_loc,
                CASE WHEN SUBSTRING(b.CONTO,1,3) IN ('410','411') THEN 1 ELSE 0 END AS is_government,
                escr.CON_ID_ESCR             AS escrow_flag,
                conto.CONTO_TYPE_ID          AS other_liab_id,
                CASE WHEN LOWER(b.CON_TYPE) IN ('deposit','min_bal','current')
                     THEN UPPER(b.CON_TYPE) END                    AS prod_class
            FROM  [ALM].[ALM].[VW_balance_rest_all]                 b  WITH (NOLOCK)
            LEFT JOIN [LIQUIDITY].[liq].[man_Client_Attr]           cli
                   ON cli.CLI_ID  = b.CLI_ID
                  AND cli.SRC_SYS = '001'
            LEFT JOIN [LIQUIDITY].[dwh].[acc_x_escrow_rest]         escr
                   ON escr.CON_ID_ESCR = b.CON_ID
                  AND escr.DT_REP      = (SELECT MAX(DT_REP)
                                          FROM [LIQUIDITY].[dwh].[acc_x_escrow_rest]
                                          WHERE DT_REP <= @dt_rep)
            LEFT JOIN [LIQUIDITY].[ratio].[man_Conto_For_LiquidityRatio]  conto
                   ON conto.CONTO = b.CONTO
                  AND @dt_rep BETWEEN conto.DT_FROM AND conto.DT_TO
            WHERE b.DT_REP      = @dt_rep
              AND b.OD_FLAG     = 1                 -- активные
              AND b.CLI_TYPE    = 'L'               -- юр-лица
              AND LOWER(b.CONTO_TYPE) = 'l'         -- пассивы
              AND ISNULL(b.CON_TYPE,'') NOT IN ('mbd','mbk','security_deal','repo')
              AND LOWER(b.ACC_ROLE) <> 'undefined'
        ),
        gv_liab AS (
            SELECT *
            FROM   base
            WHERE  acc_role IN ('liab','liab_int')      -- тело + проценты
              AND  prod_class     IS NOT NULL
              AND  is_domrf       = 0
              AND  is_loc         = 0
              AND  escrow_flag   IS NULL
              AND  is_broker      = 0
              AND  other_liab_id IS NULL
        ),
        deal_sum AS (
            SELECT
                SUM(OUT_RUB) AS OUT_RUB_PMT
            FROM   gv_liab
            GROUP  BY CON_ID
        )
        SELECT
            @dt_rep           AS dt_rep,
            N'ЮЛ'             AS addend_name,
            SUM(OUT_RUB_PMT)  AS balance_rub
        FROM   deal_sum
    ),
    fl_balance AS (
        /* ——— ФЛ через VW_alm_balance_AGG_ALMREPORT ——— */
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
    ),
    balances AS (
        SELECT * FROM ul_balance
        UNION ALL
        SELECT * FROM fl_balance
    )

    /* ---------- вставка / обновление ----------------------------- */
    MERGE liq.Liq_Balance AS tgt
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
CREATE OR ALTER PROCEDURE liq.usp_SaveOutflow
    @dt_rep       date,
    @addend_name  nvarchar(50),   -- 'ЮЛ' | 'Средства ФЛ'
    @normativ     nvarchar(50),   -- напр. 'ГВ 70'
    @amount_sum   decimal(38,2)
AS
BEGIN
    SET NOCOUNT ON;

    /* баланс должен быть занесён раньше */
    IF NOT EXISTS (SELECT 1
                   FROM   liq.Liq_Balance
                   WHERE  dt_rep      = @dt_rep
                     AND  addend_name = @addend_name)
    BEGIN
        THROW 51000, N'Сначала сохраните баланс (liq.usp_SaveBalance).', 1;
    END

    MERGE liq.Liq_Outflow AS tgt
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
CREATE OR ALTER VIEW liq.vw_Liq_Ratio
AS
SELECT
    b.dt_rep,
    b.addend_name,
    piv.[ГВ 70]      AS Ratio_GV70,
    piv.[ГВ 100]     AS Ratio_GV100,
    piv.[АЛМ stress] AS Ratio_ALM_Stress
FROM liq.Liq_Balance AS b
OUTER APPLY (
    SELECT *
    FROM (
        SELECT
            o.normativ,
            CAST( o.amount_sum /
                  NULLIF(b.balance_rub,0) AS decimal(28,8) ) AS ratio
        FROM liq.Liq_Outflow o
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

### Пример вызова

```sql
DECLARE @d date = '2025-05-13';

-- 1. фиксируем баланс
EXEC liq.usp_SaveBalance @dt_rep = @d;

-- 2. заносим оттоки (пример)
EXEC liq.usp_SaveOutflow @dt_rep=@d, @addend_name=N'ЮЛ',          @normativ=N'ГВ 70',     @amount_sum=123456.78;
EXEC liq.usp_SaveOutflow @dt_rep=@d, @addend_name=N'Средства ФЛ', @normativ=N'ГВ 70',     @amount_sum=98765.43;

-- 3. смотрим витрину
SELECT * FROM liq.vw_Liq_Ratio WHERE dt_rep = @d;
```
