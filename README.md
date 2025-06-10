/* ---------- агрегируем оттоки ---------------- */
WITH base AS (
    SELECT
        CAST(DT_REP AS date)         AS DT_REP,          -- сразу приводим к date
        CASE
            WHEN ADDEND_NAME = N'Средства ФЛ' THEN N'Средства ФЛ'
            ELSE N'ЮЛ'
        END                            AS ADDEND_NAME,
        SUM(AMOUNT_RUB_MOD)            AS AMOUNT_SUM
    FROM   [LIQUIDITY].[ratio].[VW_SH_Ratio_Agg_LVL2_Fact] WITH (NOLOCK)
    WHERE  OrganizationName = N'Банк ДОМ.РФ'
      AND  ADDEND_TYPE      = N'Оттоки'
      AND  ADDEND_DESCR    <> N'Аккредитивы'
      AND  ADDEND_NAME IN (
              N'Средства ФЛ',
              N'Средства ЮЛ',
              N'Средства Ф/О в рамках станд. продуктов',
              N'Средства Ф/О'
          )
    GROUP BY
        CAST(DT_REP AS date),
        CASE
            WHEN ADDEND_NAME = N'Средства ФЛ' THEN N'Средства ФЛ'
            ELSE N'ЮЛ'
        END
),

/* ---------- оба набора балансов в один CTE ----- */
balances AS (
    /* ЮЛ */
    SELECT
        CAST([Date] AS date)           AS DT_REP,
        [BALANCE_RUB],
        N'ЮЛ'                          AS ADDEND_NAME
    FROM   ALM_TEST.[WORK].[GroupDepositInterestsRate_UL_matur_pdr_fo] WITH (NOLOCK)
    WHERE  [TYPE]              = N'Срез'
      AND  [CLI_SUBTYPE]       = N'ЮЛ'
      AND  [TERM_GROUP]        = N'all termgroup'
      AND  [SEG_NAME]          = N'all segment'
      AND  [IS_PDR]            = N'all PDR'
      AND  [IS_FINANCE_LCR]    = N'all FINANCE_LCR'
      AND  [IS_OPTION]         = N'all option_type'
      AND  [MARGIN_TYPE]       = N'all margin'

    UNION ALL

    /* ФЛ */
    SELECT
        CAST([Date] AS date)           AS DT_REP,
        [BALANCE_RUB],
        N'Средства ФЛ'                 AS ADDEND_NAME
    FROM   ALM_TEST.[WORK].[GroupDepositInterestsRate] WITH (NOLOCK)
    WHERE  [TYPE]                 = N'Срез'
      AND  [CLI_SUBTYPE]          = N'all client'
      AND  [TERM_GROUP]           = N'all termgroup'
      AND  [SEG_NAME]             = N'all segment'
      AND  [TSEGMENTNAME]         = N'all segment'
      AND  [Spread_KeyRate_BUCKET]= N'all spread bucket'
      AND  [IS_OPTION]            = N'all option_type'
      AND  [MARGIN_TYPE]          = N'all margin'
)

/* ---------- финальный вывод -------------------- */
SELECT
    b.DT_REP,
    b.ADDEND_NAME,
    b.AMOUNT_SUM,
    bal.BALANCE_RUB
FROM   base     AS b
LEFT JOIN balances AS bal
       ON  b.DT_REP      = bal.DT_REP
       AND b.ADDEND_NAME = bal.ADDEND_NAME
ORDER BY b.DT_REP DESC;
