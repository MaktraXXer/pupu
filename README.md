/* ===================================================================
   ПАРАМЕТРОВ НЕ НУЖНО: даты подтягиваются из самих оттоков
   -------------------------------------------------------------------
 =================================================================== */

/* -----------------------------------------------------------
   1) Оттоки (как были) ― берём только нужные категории
----------------------------------------------------------- */
WITH base AS (
    SELECT
        CAST(DT_REP AS date)                         AS DT_REP,
        CASE WHEN ADDEND_NAME = N'Средства ФЛ'
             THEN N'Средства ФЛ'
             ELSE N'ЮЛ'
        END                                          AS ADDEND_NAME,
        SUM(AMOUNT_RUB_MOD)                          AS AMOUNT_SUM
    FROM   [LIQUIDITY].[ratio].[VW_SH_Ratio_Agg_LVL2_Fact]  WITH (NOLOCK)
    WHERE  OrganizationName = N'Банк ДОМ.РФ'
      AND  ADDEND_TYPE      = N'Оттоки'
      AND  ADDEND_DESCR    <> N'Аккредитивы'
      AND  ADDEND_NAME IN ( N'Средства ФЛ',
                            N'Средства ЮЛ',
                            N'Средства Ф/О в рамках станд. продуктов',
                            N'Средства Ф/О')
    GROUP BY
        CAST(DT_REP AS date),
        CASE WHEN ADDEND_NAME = N'Средства ФЛ'
             THEN N'Средства ФЛ'
             ELSE N'ЮЛ'
        END
),

/* -----------------------------------------------------------
   2) Список дат из оттоков (чтобы ЮЛ и ФЛ считались
      ТОЛЬКО на эти дни)
----------------------------------------------------------- */
dt_list AS (
    SELECT DISTINCT DT_REP FROM base
),

/* -----------------------------------------------------------
   3) «Правильный» срез пассивов ЮЛ  (тот, который
      мы уже собрали раньше, но теперь ― на диапазон дат)
----------------------------------------------------------- */
ul_liab_base AS (
    SELECT
        b.DT_REP,
        b.CON_ID,
        b.CLI_ID,
        b.ACC_NO,
        b.PROD_ID,
        b.CON_TYPE,
        b.CONTO,
        b.CUR,
        ABS(b.OUT_RUB)       AS OUT_RUB,
        ABS(b.OUT_CUR)       AS OUT_CUR,
        LOWER(b.ACC_ROLE)    AS acc_role,

        cli.IS_FINANCE_LCR   AS is_finance,
        cli.IS_SSV           AS is_ssv,
        cli.ISDOMRF          AS is_domrf,

        CASE WHEN b.PROD_ID IN (398,399,400) THEN 1 ELSE 0 END             AS is_broker,
        CASE WHEN LOWER(b.CON_TYPE) = 'loc'          THEN 1 ELSE 0 END      AS is_loc,
        CASE WHEN SUBSTRING(b.CONTO,1,3) IN ('410','411') THEN 1 ELSE 0 END AS is_government,

        escr.CON_ID_ESCR     AS escrow_flag,
        conto.CONTO_TYPE_ID  AS other_liab_id,

        CASE WHEN LOWER(b.CON_TYPE) IN ('deposit','min_bal','current')
             THEN UPPER(b.CON_TYPE) END                                    AS prod_class
    FROM   [ALM].[ALM].[VW_balance_rest_all]                b  WITH (NOLOCK)
    LEFT JOIN [LIQUIDITY].[liq].[man_Client_Attr]           cli
           ON cli.CLI_ID  = b.CLI_ID
          AND cli.SRC_SYS = '001'
    LEFT JOIN [LIQUIDITY].[dwh].[acc_x_escrow_rest]         escr
           ON escr.CON_ID_ESCR = b.CON_ID
          AND escr.DT_REP      = (SELECT MAX(DT_REP)
                                  FROM [LIQUIDITY].[dwh].[acc_x_escrow_rest]
                                  WHERE DT_REP <= b.DT_REP)
    LEFT JOIN [LIQUIDITY].[ratio].[man_Conto_For_LiquidityRatio]  conto
           ON conto.CONTO = b.CONTO
          AND b.DT_REP BETWEEN conto.DT_FROM AND conto.DT_TO
    /* ---------- основные фильтры ---------- */
    WHERE b.DT_REP IN (SELECT DT_REP FROM dt_list)   -- только нужные даты
      AND b.OD_FLAG        = 1        -- активные
      AND b.CLI_TYPE       = 'L'      -- ЮЛ
      AND LOWER(b.CONTO_TYPE) = 'l'   -- пассивы
      AND ISNULL(b.CON_TYPE,'') NOT IN ('mbd','mbk','security_deal','repo')
      AND LOWER(b.ACC_ROLE)  <> 'undefined'
),

gv_liab AS (
    SELECT *
    FROM   ul_liab_base
    WHERE  acc_role IN ('liab','liab_int')          -- тело + проценты
      AND  prod_class IS NOT NULL
      AND  is_domrf      = 0
      AND  is_loc        = 0
      AND  escrow_flag  IS NULL
      AND  is_broker     = 0
      AND  other_liab_id IS NULL
),

ul_bal AS (
    SELECT
        DT_REP,
        SUM(OUT_RUB)           AS BALANCE_RUB,
        N'ЮЛ'                  AS ADDEND_NAME
    FROM   gv_liab
    GROUP  BY DT_REP
),

/* -----------------------------------------------------------
   4) Пассивы ФЛ (ограничены теми же датами)
----------------------------------------------------------- */
fl_bal AS (
    SELECT
        CAST(dt_rep AS date)                     AS DT_REP,
        SUM(ISNULL(sOUT_RUB, 0))*1000            AS BALANCE_RUB,
        N'Средства ФЛ'                           AS ADDEND_NAME
    FROM   ALM.ALM.VW_alm_balance_AGG_ALMREPORT  WITH (NOLOCK)
    WHERE  CAST(dt_rep AS date) IN (SELECT DT_REP FROM dt_list)
      AND  AP             = N'Пассив'
      AND  section_name   IN ('До востребования','Срочные ','Накопительный счёт')
      AND  block_name     = N'Привлечение ФЛ'
      AND  TSegmentName   IN ('ДЧБО','Розничный Бизнес')
    GROUP BY CAST(dt_rep AS date)
),

/* -----------------------------------------------------------
   5) Объединяем баланс ЮЛ + ФЛ
----------------------------------------------------------- */
balances AS (
    SELECT * FROM ul_bal
    UNION ALL
    SELECT * FROM fl_bal
)

/* -----------------------------------------------------------
   6) Итоговый вывод
----------------------------------------------------------- */
SELECT
    b.DT_REP,
    b.ADDEND_NAME,
    b.AMOUNT_SUM,
    bal.BALANCE_RUB,
    ABS(b.AMOUNT_SUM) / NULLIF(ABS(bal.BALANCE_RUB),0)  AS [ГВ 70]
FROM   base      AS b
LEFT  JOIN balances AS bal
       ON  b.DT_REP      = bal.DT_REP
       AND b.ADDEND_NAME = bal.ADDEND_NAME
ORDER BY b.DT_REP DESC, b.ADDEND_NAME;
