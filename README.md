/* Период, который точно есть в обеих витринах */
DECLARE
    @d_from date = '2025-05-01',
    @d_to   date = '2025-06-24';

/* ------------ 1. Коллеги: vitrina dtStartDeal ------------ */
WITH cte_col AS (
    SELECT  [Date],
            SUM(BALANCE_RUB) AS bal_col
    FROM    LIQUIDITY.liq.GroupDepositInterestsRate_dtStart
    WHERE   [Date] BETWEEN @d_from AND @d_to
            AND [TYPE]        = 'Начало'
            AND CLI_SUBTYPE   = 'ЮЛ'
            AND Spread_KeyRate= 'all spread bucket'
    GROUP BY [Date]
)

/* ------------ 2. Вы: vitrina UL_matur_pdr_fo ------------- */
, cte_you AS (
    SELECT  [Date],
            SUM(BALANCE_RUB) AS bal_you
    FROM    ALM_TEST.WORK.GroupDepositInterestsRate_UL_matur_pdr_fo
    WHERE   [Date] BETWEEN @d_from AND @d_to
            AND [TYPE]          = 'Начало день ко дню'
            AND CLI_SUBTYPE     = 'ЮЛ'
            AND IS_PDR          = 'all PDR'
            AND IS_FINANCE_LCR  = 'all FINANCE_LCR'
    GROUP BY [Date]
)

/* ------------ 3. Итоговое сравнение ---------------------- */
SELECT  COALESCE(c.[Date], y.[Date])            AS [Date],
        ISNULL(c.bal_col, 0)                    AS bal_colleague,
        ISNULL(y.bal_you, 0)                    AS bal_yours,
        ISNULL(y.bal_you,0) - ISNULL(c.bal_col,0) AS diff
FROM    cte_col c
FULL JOIN cte_you y
       ON y.[Date] = c.[Date]
ORDER BY [Date];
