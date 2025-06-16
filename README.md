/* -----------------------------------------------------------
   Параметр среза
----------------------------------------------------------- */
DECLARE @dtRep DATE = '2025-06-07';      -- <-- нужная дата

/* -----------------------------------------------------------
   1. Базовая выборка всех пассивов юр-лиц за дату
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
        /* оригинальные остатки */
        ABS(b.OUT_RUB) AS OUT_RUB,
        ABS(b.OUT_CUR) AS OUT_CUR,
        LOWER(b.ACC_ROLE)  AS acc_role,

        /* признаки, которые процедура выводила позже */
        cli.IS_FINANCE_LCR       AS is_finance,
        cli.IS_SSV               AS is_ssv,
        cli.ISDOMRF              AS is_domrf,
        CASE WHEN b.PROD_ID IN (398,399,400) THEN 1 ELSE 0 END  AS is_broker,
        CASE WHEN LOWER(b.CON_TYPE)='loc'          THEN 1 ELSE 0 END AS is_loc,
        CASE WHEN SUBSTRING(b.CONTO,1,3) IN ('410','411') THEN 1 ELSE 0 END AS is_government,
        escr.CON_ID_ESCR         AS escrow_flag,
        conto.CONTO_TYPE_ID      AS other_liab_id,
        CASE WHEN LOWER(b.CON_TYPE) IN ('deposit','min_bal','current')
             THEN UPPER(b.CON_TYPE) END                        AS prod_class   -- DEPOSIT | MIN_BAL | CURRENT
    FROM  [ALM].[ALM].[VW_balance_rest_all]           b  WITH (NOLOCK)

    /* признаки клиента */
    LEFT JOIN [LIQUIDITY].[liq].[man_Client_Attr]     cli
           ON cli.CLI_ID  = b.CLI_ID
          AND cli.SRC_SYS = '001'

    /* эскроу-сделки */
    LEFT JOIN [LIQUIDITY].[dwh].[acc_x_escrow_rest]   escr
           ON escr.CON_ID_ESCR = b.CON_ID
          AND escr.DT_REP      = (SELECT MAX(DT_REP)
                                  FROM [LIQUIDITY].[dwh].[acc_x_escrow_rest]
                                  WHERE DT_REP <= @dtRep)

    /* спец-счета (OTHER_LIAB_TYPE_ID) */
    LEFT JOIN [LIQUIDITY].[ratio].[man_Conto_For_LiquidityRatio]  conto
           ON conto.CONTO = b.CONTO
          AND @dtRep BETWEEN conto.DT_FROM AND conto.DT_TO

    WHERE b.DT_REP      = @dtRep
      AND b.OUT_RUB     IS NOT NULL
      AND LOWER(b.ACC_ROLE) <> 'undefined'
      AND ISNULL(b.CON_TYPE,'') NOT IN ('mbd','mbk','security_deal','repo')
      AND LOWER(b.CONTO_TYPE)  = 'l'      -- пассивы
      AND b.OD_FLAG            = 1        -- только действующие
      AND b.CLI_TYPE           = 'L'      -- юр-лица
)

/* -----------------------------------------------------------
   2. Отфильтровываем «старый ГВ» и суммируем principal+interest
----------------------------------------------------------- */
, gv_liab AS (
    SELECT *
    FROM   base
    WHERE  /* --- нужные отборы --- */
           is_domrf       = 0         -- без ВГО
       AND is_loc         = 0         -- без аккредитивов
       AND escrow_flag   IS NULL      -- без эскроу
       AND is_broker      = 0         -- без брокерских счетов
       AND is_government  = 0         -- без бюджетных средств
       AND other_liab_id IS NULL      -- без спец-счётов
       /* --- продуктовая корзина (депозиты / НСО / текущие) --- */
       AND prod_class IS NOT NULL
       /* --- две целевые группы юр-лиц --- */
       AND (
             (is_finance = 0 AND is_ssv = 0)     -- обычные ЮЛ
          OR (is_finance = 1)                    -- финорганизации (стандартные продукты)
           )
       /* роли, которые участвуют в суммировании тела + процентов */
       AND acc_role IN ('liab','liab_int')
)

/* -----------------------------------------------------------
   3. Агрегируем по сделке (CON_ID) — получаем OUT_RUB_PMT / OUT_CUR_PMT
----------------------------------------------------------- */
SELECT
    MIN(DT_REP)            AS DT_REP,
    CON_ID,
    MIN(CLI_ID)            AS CLI_ID,
    MIN(ACC_NO)            AS ACC_NO,
    MIN(PROD_ID)           AS PROD_ID,
    MIN(prod_class)        AS PRODUCT_CLASS,   -- DEPOSIT | MIN_BAL | CURRENT
    MIN(CON_TYPE)          AS CON_TYPE_RAW,
    MIN(CONTO)             AS CONTO,
    MIN(CUR)               AS CUR,
    /*  суммы: тело + начисленные проценты  */
    SUM(OUT_RUB)           AS OUT_RUB_PMT,
    SUM(OUT_CUR)           AS OUT_CUR_PMT
FROM   gv_liab
GROUP  BY CON_ID
ORDER  BY PRODUCT_CLASS, ACC_NO;
