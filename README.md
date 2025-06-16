/* -----------------------------------------------------------
   Параметры
----------------------------------------------------------- */
DECLARE @dtRep DATE = '2025-06-07';      -- ← подставь нужную дату

/* -----------------------------------------------------------
   1. Базовый срез пассивов юр-лиц
----------------------------------------------------------- */
WITH base AS (
    SELECT
        b.DT_REP,
        b.CON_ID,
        b.CLI_ID,
        b.ACC_NO,
        b.PROD_ID,
        b.CON_TYPE,
        b.CONTO,
        b.CUR,
        ABS(b.OUT_RUB) AS OUT_RUB,
        ABS(b.OUT_CUR) AS OUT_CUR,
        LOWER(b.ACC_ROLE)  AS acc_role,

        /* клиентские признаки */
        cli.IS_FINANCE_LCR       AS is_finance,
        cli.IS_SSV               AS is_ssv,
        cli.ISDOMRF              AS is_domrf,

        /* ещё фильтры */
        CASE WHEN b.PROD_ID IN (398,399,400) THEN 1 ELSE 0 END  AS is_broker,
        CASE WHEN LOWER(b.CON_TYPE) = 'loc'                     THEN 1 ELSE 0 END AS is_loc,
        CASE WHEN SUBSTRING(b.CONTO,1,3) IN ('410','411')       THEN 1 ELSE 0 END AS is_government,
        escr.CON_ID_ESCR         AS escrow_flag,
        conto.CONTO_TYPE_ID      AS other_liab_id,

        /* продуктовая корзина для ГВ */
        CASE WHEN LOWER(b.CON_TYPE) IN ('deposit','min_bal','current')
             THEN UPPER(b.CON_TYPE) END                        AS prod_class
    FROM  [ALM].[ALM].[VW_balance_rest_all]                 b  WITH (NOLOCK)

    /* признаки клиента */
    LEFT JOIN [LIQUIDITY].[liq].[man_Client_Attr]           cli
           ON cli.CLI_ID  = b.CLI_ID
          AND cli.SRC_SYS = '001'

    /* эскроу-счета */
    LEFT JOIN [LIQUIDITY].[dwh].[acc_x_escrow_rest]         escr
           ON escr.CON_ID_ESCR = b.CON_ID
          AND escr.DT_REP      = (SELECT MAX(DT_REP)
                                  FROM [LIQUIDITY].[dwh].[acc_x_escrow_rest]
                                  WHERE DT_REP <= @dtRep)

    /* спец-счета (OTHER_LIAB_TYPE_ID) */
    LEFT JOIN [LIQUIDITY].[ratio].[man_Conto_For_LiquidityRatio]  conto
           ON conto.CONTO = b.CONTO
          AND @dtRep BETWEEN conto.DT_FROM AND conto.DT_TO

    WHERE b.DT_REP  = @dtRep
      AND b.OD_FLAG = 1                     -- активные
      AND b.CLI_TYPE = 'L'                  -- юр-лица
      AND LOWER(b.CONTO_TYPE) = 'l'         -- пассивы
      AND ISNULL(b.CON_TYPE,'') NOT IN ('mbd','mbk','security_deal','repo')
      AND LOWER(b.ACC_ROLE) <> 'undefined'
)

/* -----------------------------------------------------------
   2. «Старый ГВ»: срез + нужные роли
----------------------------------------------------------- */
, gv_liab AS (
    SELECT *
    FROM   base
    WHERE  acc_role IN ('liab','liab_int')            -- тело + проценты
      AND prod_class IS NOT NULL                      -- депозиты/НСО/текущие
      AND (
            (is_finance = 0 AND is_ssv = 0)           -- обычные ЮЛ
         OR (is_finance = 1)                          -- финорганизации
          )
      AND is_domrf      = 0
      AND is_loc        = 0
      AND escrow_flag  IS NULL
      AND is_broker     = 0
      AND is_government = 0
      AND other_liab_id IS NULL
)

/* -----------------------------------------------------------
   3. Остаток сделки = тело + начисленные проценты
----------------------------------------------------------- */
, deal_sum AS (
    SELECT
        MIN(DT_REP)  AS DT_REP,
        CON_ID,
        SUM(OUT_RUB) AS OUT_RUB_PMT,
        SUM(OUT_CUR) AS OUT_CUR_PMT
    FROM   gv_liab
    GROUP  BY CON_ID
)

/* -----------------------------------------------------------
   4. Итоговая сумма на дату (DT_REP)
----------------------------------------------------------- */
SELECT
    DT_REP,
    SUM(OUT_RUB_PMT) AS TOTAL_OUT_RUB_PMT,
    SUM(OUT_CUR_PMT) AS TOTAL_OUT_CUR_PMT   -- в оригинальной валюте
FROM   deal_sum
GROUP  BY DT_REP;
