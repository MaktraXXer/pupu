/* ===================================================================
   0) ОТТОКИ  – базовый набор дат + суммы
   -------------------------------------------------------------------
   (это единственный раз, когда мы лезем в VW_SH_Ratio_Agg_LVL2_Fact)
=================================================================== */
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

/* ===================================================================
   1) БАЛАНСЫ  – одним UNION-ом сразу ЮЛ + ФЛ
   -------------------------------------------------------------------
   – датами ограничиваемся «как в base» → меньше строк с трафика
   – ЮЛ агрегируем сразу, без промежуточных выборок
=================================================================== */
balances AS (
    SELECT
        b.DT_REP,
        b.ADDEND_NAME,
        SUM(b.BALANCE_RUB)              AS BALANCE_RUB
    FROM (
        /* ---------- ЮЛ (сразу с фильтрами) ----------------------- */
        SELECT
            bl.DT_REP,
            N'ЮЛ'                       AS ADDEND_NAME,
            SUM(ABS(bl.OUT_RUB))        AS BALANCE_RUB
        FROM   [ALM].[ALM].[VW_balance_rest_all] bl WITH (NOLOCK)

        /* --- даты только те, что есть в base -------------------- */
        JOIN  (SELECT DISTINCT DT_REP FROM base) d
              ON bl.DT_REP = d.DT_REP

        /* --- основные фильтры «правильного» пассива ЮЛ ---------- */
        WHERE bl.OD_FLAG          = 1                     -- активные
          AND bl.CLI_TYPE         = 'L'                   -- юр-лица
          AND LOWER(bl.CONTO_TYPE)= 'l'                   -- пассивы
          AND ISNULL(bl.CON_TYPE,'') NOT IN ('mbd','mbk','security_deal','repo')
          AND LOWER(bl.ACC_ROLE)  IN ('liab','liab_int')  -- тело + проценты
          AND bl.PROD_ID NOT IN (398,399,400)             -- !broker
          AND LOWER(bl.CON_TYPE) <> 'loc'                 -- !LOC
          AND SUBSTRING(bl.CONTO,1,3) NOT IN ('410','411')-- !гос-счета
          /* -- escrow & прочие «другие обязательства» ------------- */
          AND NOT EXISTS (
                  SELECT 1
                  FROM [LIQUIDITY].[dwh].[acc_x_escrow_rest] e
                  WHERE e.CON_ID_ESCR = bl.CON_ID
                    AND e.DT_REP = (
                        SELECT MAX(DT_REP)
                        FROM [LIQUIDITY].[dwh].[acc_x_escrow_rest]
                        WHERE DT_REP <= bl.DT_REP) )
          AND NOT EXISTS (
                  SELECT 1
                  FROM [LIQUIDITY].[ratio].[man_Conto_For_LiquidityRatio] c
                  WHERE c.CONTO = bl.CONTO
                    AND bl.DT_REP BETWEEN c.DT_FROM AND c.DT_TO
                    AND c.CONTO_TYPE_ID IS NOT NULL )
        GROUP BY bl.DT_REP

        UNION ALL

        /* ---------- ФЛ ------------------------------------------ */
        SELECT
            CAST(fl.dt_rep AS date)     AS DT_REP,
            N'Средства ФЛ'              AS ADDEND_NAME,
            SUM(ISNULL(fl.sOUT_RUB,0))*1000 AS BALANCE_RUB
        FROM   ALM.ALM.VW_alm_balance_AGG_ALMREPORT  fl WITH (NOLOCK)
        JOIN  (SELECT DISTINCT DT_REP FROM base) d
              ON CAST(fl.dt_rep AS date) = d.DT_REP
        WHERE  fl.AP           = N'Пассив'
          AND  fl.section_name IN ('До востребования','Срочные ','Накопительный счёт')
          AND  fl.block_name   = N'Привлечение ФЛ'
          AND  fl.TSegmentName IN ('ДЧБО','Розничный Бизнес')
        GROUP BY CAST(fl.dt_rep AS date)
    ) b
    GROUP BY b.DT_REP, b.ADDEND_NAME
)

/* ===================================================================
   2) ФИНАЛ: дата / категория / оттоки / остаток / показатель «ГВ 70»
=================================================================== */
SELECT
    base.DT_REP,
    base.ADDEND_NAME,
    base.AMOUNT_SUM,
    bal.BALANCE_RUB,
    ABS(base.AMOUNT_SUM) / NULLIF(ABS(bal.BALANCE_RUB),0)  AS [ГВ 70]
FROM   base
LEFT  JOIN balances bal
       ON  base.DT_REP      = bal.DT_REP
       AND base.ADDEND_NAME = bal.ADDEND_NAME
ORDER BY base.DT_REP DESC, base.ADDEND_NAME;
