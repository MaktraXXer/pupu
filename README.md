CREATE OR ALTER PROCEDURE liq.usp_SaveBalance
    @dt_rep date
AS
BEGIN
    SET NOCOUNT ON;

    /* ---------- ЮЛ: «старый ГВ» через VW_balance_rest_all ---------- */
    ;WITH base AS (          -- 1-й CTE
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
        FROM  ALM.ALM.VW_balance_rest_all                    b WITH (NOLOCK)
        LEFT JOIN LIQUIDITY.liq.man_Client_Attr              cli
               ON cli.CLI_ID  = b.CLI_ID
              AND cli.SRC_SYS = '001'
        LEFT JOIN LIQUIDITY.dwh.acc_x_escrow_rest            escr
               ON escr.CON_ID_ESCR = b.CON_ID
              AND escr.DT_REP      =
                  (SELECT MAX(DT_REP)
                   FROM LIQUIDITY.dwh.acc_x_escrow_rest
                   WHERE DT_REP <= @dt_rep)
        LEFT JOIN LIQUIDITY.ratio.man_Conto_For_LiquidityRatio conto
               ON conto.CONTO = b.CONTO
              AND @dt_rep BETWEEN conto.DT_FROM AND conto.DT_TO
        WHERE b.DT_REP      = @dt_rep
          AND b.OD_FLAG     = 1
          AND b.CLI_TYPE    = 'L'
          AND LOWER(b.CONTO_TYPE) = 'l'
          AND ISNULL(b.CON_TYPE,'') NOT IN ('mbd','mbk','security_deal','repo')
          AND LOWER(b.ACC_ROLE) <> 'undefined'
    ),
    gv_liab AS (             -- 2-й CTE
        SELECT *
        FROM   base
        WHERE  acc_role IN ('liab','liab_int')
          AND  prod_class     IS NOT NULL
          AND  is_domrf       = 0
          AND  is_loc         = 0
          AND  escrow_flag   IS NULL
          AND  is_broker      = 0
          AND  other_liab_id IS NULL
    ),
    deal_sum AS (            -- 3-й CTE
        SELECT SUM(OUT_RUB) AS OUT_RUB_PMT
        FROM   gv_liab
        GROUP  BY CON_ID
    ),
    ul_balance AS (          -- 4-й CTE
        SELECT
            @dt_rep           AS dt_rep,
            N'ЮЛ'             AS addend_name,
            SUM(OUT_RUB_PMT)  AS balance_rub
        FROM   deal_sum
    ),

    /* ---------- ФЛ через VW_alm_balance_AGG_ALMREPORT ------------ */
    fl_balance AS (          -- 5-й CTE (параллельно)
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
    balances AS (            -- итоговый CTE
        SELECT * FROM ul_balance
        UNION ALL
        SELECT * FROM fl_balance
    )

    /* ---------- MERGE в таблицу ---------------------------------- */
    MERGE liq.Liq_Balance AS tgt
    USING balances        AS src
      ON  tgt.dt_rep      = src.dt_rep
     AND  tgt.addend_name = src.addend_name
    WHEN MATCHED THEN
         UPDATE SET balance_rub = src.balance_rub
    WHEN NOT MATCHED THEN
         INSERT (dt_rep, addend_name, balance_rub)
         VALUES (src.dt_rep, src.addend_name, src.balance_rub);
END
GO
