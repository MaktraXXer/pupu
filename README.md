USE ALM_TEST;
GO

CREATE OR ALTER VIEW [WORK].[vw_ProlongationAnalysis_OpenAndClosed_BalanceRub_WithRates]
AS
/*
  Это представление агрегирует данные по депозитам, начиная с 2024 года.
  
  Для открытых депозитов (по DT_OPEN):
    - Отбираются депозиты, открытые в месяце, которые прожили ≥ 10 дней.
    - MATUR = DATEDIFF(DAY, DT_OPEN, DT_CLOSE_PLAN); ConvertedRate рассчитывается через функцию
          LIQUIDITY.liq.fnc_IntRate(RATE, NEW_CONVENTION_NAME, 'monthly', MATUR, 1)
    - Информация из LIQUIDITY.[liq].man_CONVENTION используется для получения NEW_CONVENTION_NAME.
    - По DaysLived (DATEDIFF(DAY, DT_OPEN, DT_CLOSE)) определяется бакет срочности (TermBucket)
      через таблицу [man_TermGroup] (поле TERM_GROUP).
    - Через LEFT JOIN с [conrel_prolongations] определяется число пролонгаций (ProlongCount).
      Если пролонгация есть (ProlongCount > 0), для первого звена (old1) рассчитывается PrevConvertedRate и берётся PrevBalance.
    - Агрегируются показатели по открытым депозитам (CTE OpenAggregated), где вычисляются:
         • OpenedDeals, Summ_BalanceRub;
         • По пролонгациям: Count_Prolong, Count_NewNoProlong, Count_1yProlong, Count_2yProlong, Count_3plusProlong;
           суммы: Sum_ProlongRub, Sum_NewNoProlong, Sum_1yProlong_Rub, Sum_2yProlong_Rub, Sum_3plusProlong_Rub;
         • Взвешенные суммы: Sum_RateWeighted = SUM(BALANCE_RUB * ConvertedRate) для всех,
           Sum_RateWeighted_NewNoProlong для депозитов без пролонгации,
           Sum_RateWeighted_1y, Sum_RateWeighted_2y, Sum_RateWeighted_3plus для соответствующих групп,
           а также Sum_RateWeighted_Prolong = SUM(CASE WHEN ProlongCount > 0 THEN BALANCE_RUB * ConvertedRate ELSE 0 END).
         • Для предыдущих депозитов – Sum_PreviousBalance и Sum_PreviousRateWeighted.
    - Из этих агрегатов можно вычислить средневзвешенные ставки:
         WeightedRate_All = Sum_RateWeighted / Summ_BalanceRub,
         WeightedRate_NewNoProlong = Sum_RateWeighted_NewNoProlong / Sum_NewNoProlong,
         WeightedRate_1y = Sum_RateWeighted_1y / Sum_1yProlong_Rub,
         WeightedRate_2y = Sum_RateWeighted_2y / Sum_2yProlong_Rub,
         WeightedRate_3plus = Sum_RateWeighted_3plus / Sum_3plusProlong_Rub,
         WeightedRate_AllProlong = Sum_RateWeighted_Prolong / Sum_ProlongRub,
         WeightedRate_Previous = Sum_PreviousRateWeighted / Sum_PreviousBalance.
  
  Для закрытых депозитов (по DT_CLOSE):
    - Аналогично отбираются депозиты, рассчитывается MATUR и ConvertedRate, определяется бакет срочности,
      агрегируются ClosedDeals, Summ_ClosedBalanceRub и Sum_RateWeighted_Closed, а затем
      WeightedRate_Closed = Sum_RateWeighted_Closed / Summ_ClosedBalanceRub.
      
  Итог – агрегаты по открытым и закрытым депозитам объединяются через FULL OUTER JOIN по ключу:
         (MonthEnd, SegmentGrouping, CurrencyGrouping, TermBucketGrouping).
  Сегментация осуществляется по MonthEnd, SegmentGrouping (с перекодировкой), CurrencyGrouping и TermBucketGrouping.
  Итоговые значения заменяются через ISNULL/COALESCE.
*/
WITH
-----------------------------
-- 0) Генерация месяцев (MonthEnd)
-----------------------------
cteMonths AS (
    SELECT CONVERT(date, '2024-01-31') AS MonthEnd
    UNION ALL
    SELECT EOMONTH(DATEADD(MONTH, 1, MonthEnd))
    FROM cteMonths
    WHERE EOMONTH(DATEADD(MONTH, 1, MonthEnd)) <= '2025-02-28'
),
-----------------------------
-- A1) Открытые депозиты с расчетом ConvertedRate
-----------------------------
DealsInMonthRates AS (
    SELECT
         M.MonthEnd,
         CAST(dc.CON_ID AS BIGINT) AS CON_ID,
         dc.SEG_NAME,
         dc.CUR,
         dc.DT_OPEN,
         dc.BALANCE_RUB,
         dc.RATE,
         DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE_PLAN) AS MATUR,
         conv.NEW_CONVENTION_NAME,
         DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE) AS DaysLived,
         LIQUIDITY.liq.fnc_IntRate(dc.RATE, conv.NEW_CONVENTION_NAME, 'monthly',
                                    DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE_PLAN), 1) AS ConvertedRate
    FROM cteMonths M
    JOIN [LIQUIDITY].[liq].[DepositContract_all] dc
         ON dc.DT_OPEN >= DATEADD(DAY, 1, EOMONTH(M.MonthEnd, -1))
        AND dc.DT_OPEN < DATEADD(DAY, 1, M.MonthEnd)
        AND dc.CLI_SUBTYPE = 'INDIV'
        AND dc.PROD_NAME   != 'ЭскроУ'
        AND dc.DT_CLOSE_PLAN != '4444-01-01'
        AND DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE) >= 10
    JOIN LIQUIDITY.[liq].man_CONVENTION conv
         ON dc.CONVENTION = conv.CONVENTION_NAME
),
-----------------------------
-- A2) Присоединяем бакеты срочности для открытых депозитов
-----------------------------
DealsInMonthBucket AS (
    SELECT
         dmr.MonthEnd,
         dmr.CON_ID,
         dmr.SEG_NAME,
         dmr.CUR,
         dmr.DT_OPEN,
         dmr.BALANCE_RUB,
         dmr.DaysLived,
         dmr.ConvertedRate,
         dmr.MATUR,
         tg.TERM_GROUP AS TermBucket
    FROM DealsInMonthRates dmr
    LEFT JOIN [ALM_TEST].[WORK].[man_TermGroup] tg
         ON dmr.DaysLived >= tg.TERM_FROM
        AND dmr.DaysLived <= tg.TERM_TO
),
-----------------------------
-- A3) Определяем число пролонгаций и получаем данные предыдущего депозита
-----------------------------
DealsWithProlong AS (
    SELECT
         dmb.MonthEnd,
         dmb.CON_ID,
         dmb.SEG_NAME,
         dmb.CUR,
         dmb.DT_OPEN,
         dmb.BALANCE_RUB,
         dmb.TermBucket,
         dmb.ConvertedRate,
         CASE
           WHEN p1.CON_ID IS NULL OR old1.CON_ID IS NULL THEN 0
           WHEN p2.CON_ID IS NULL OR old2.CON_ID IS NULL THEN 1
           WHEN p3.CON_ID IS NULL OR old3.CON_ID IS NULL THEN 2
           ELSE 3
         END AS ProlongCount,
         CASE WHEN p1.CON_ID IS NOT NULL AND old1.CON_ID IS NOT NULL 
              THEN LIQUIDITY.liq.fnc_IntRate(old1.RATE, conv_prev.NEW_CONVENTION_NAME, 'monthly',
                                              DATEDIFF(DAY, old1.DT_OPEN, old1.DT_CLOSE_PLAN), 1)
              ELSE NULL END AS PrevConvertedRate,
         CASE WHEN p1.CON_ID IS NOT NULL AND old1.CON_ID IS NOT NULL 
              THEN old1.BALANCE_RUB
              ELSE 0 END AS PrevBalance
    FROM DealsInMonthBucket dmb
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p1
         ON p1.CON_ID = dmb.CON_ID
        AND p1.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old1
         ON old1.CON_ID = p1.CON_ID_REL
        AND DATEDIFF(DAY, old1.DT_OPEN, old1.DT_CLOSE) >= 10
    LEFT JOIN LIQUIDITY.[liq].man_CONVENTION conv_prev
         ON old1.CONVENTION = conv_prev.CONVENTION_NAME
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p2
         ON p2.CON_ID = old1.CON_ID
        AND p2.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old2
         ON old2.CON_ID = p2.CON_ID_REL
        AND DATEDIFF(DAY, old2.DT_OPEN, old2.DT_CLOSE) >= 10
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p3
         ON p3.CON_ID = old2.CON_ID
        AND p3.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old3
         ON old3.CON_ID = p3.CON_ID_REL
        AND DATEDIFF(DAY, old3.DT_OPEN, old3.DT_CLOSE) >= 10
),
-----------------------------
-- A4) Агрегируем показатели по открытым депозитам
-----------------------------
OpenAggregated AS (
    SELECT 
         MonthEnd,
         CASE 
           WHEN SEG_NAME = 'Розничный бизнес' THEN N'Розница'
           WHEN SEG_NAME = 'ДЧБО' THEN N'ЧБО'
           ELSE N'Без сегментации'
         END AS SegmentGrouping,
         CUR AS CurrencyGrouping,
         TermBucket AS TermBucketGrouping,
         COUNT(*) AS OpenedDeals,
         SUM(BALANCE_RUB) AS Summ_BalanceRub,
         SUM(CASE WHEN ProlongCount > 0 THEN 1 ELSE 0 END) AS Count_Prolong,
         SUM(CASE WHEN ProlongCount = 0 THEN 1 ELSE 0 END) AS Count_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN 1 ELSE 0 END) AS Count_1yProlong,
         SUM(CASE WHEN ProlongCount = 2 THEN 1 ELSE 0 END) AS Count_2yProlong,
         SUM(CASE WHEN ProlongCount >= 3 THEN 1 ELSE 0 END) AS Count_3plusProlong,
         SUM(CASE WHEN ProlongCount > 0 THEN BALANCE_RUB ELSE 0 END) AS Sum_ProlongRub,
         SUM(CASE WHEN ProlongCount = 0 THEN BALANCE_RUB ELSE 0 END) AS Sum_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN BALANCE_RUB ELSE 0 END) AS Sum_1yProlong_Rub,
         SUM(CASE WHEN ProlongCount = 2 THEN BALANCE_RUB ELSE 0 END) AS Sum_2yProlong_Rub,
         SUM(CASE WHEN ProlongCount >= 3 THEN BALANCE_RUB ELSE 0 END) AS Sum_3plusProlong_Rub,
         SUM(BALANCE_RUB * ConvertedRate) AS Sum_RateWeighted,
         SUM(CASE WHEN ProlongCount = 0 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_1y,
         SUM(CASE WHEN ProlongCount = 2 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_2y,
         SUM(CASE WHEN ProlongCount >= 3 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_3plus,
         SUM(CASE WHEN ProlongCount > 0 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_Prolong,
         SUM(CASE WHEN ProlongCount > 0 THEN PrevBalance ELSE 0 END) AS Sum_PreviousBalance,
         SUM(CASE WHEN ProlongCount > 0 THEN PrevBalance * PrevConvertedRate ELSE 0 END) AS Sum_PreviousRateWeighted
    FROM DealsWithProlong
    GROUP BY CUBE (
         MonthEnd,
         CASE 
           WHEN SEG_NAME = 'Розничный бизнес' THEN N'Розница'
           WHEN SEG_NAME = 'ДЧБО' THEN N'ЧБО'
           ELSE N'Без сегментации'
         END,
         CUR,
         TermBucket
    )
),
-----------------------------
-- A5) Расчет средневзвешенных ставок для открытых депозитов
-----------------------------
OpenWeightedRates AS (
    SELECT 
         MonthEnd,
         SegmentGrouping,
         CurrencyGrouping,
         TermBucketGrouping,
         CASE WHEN SUM(Summ_BalanceRub) > 0 THEN SUM(Sum_RateWeighted) / SUM(Summ_BalanceRub) ELSE 0 END AS WeightedRate_All,
         CASE WHEN SUM(Sum_NewNoProlong) > 0 THEN SUM(Sum_RateWeighted_NewNoProlong) / SUM(Sum_NewNoProlong) ELSE 0 END AS WeightedRate_NewNoProlong,
         CASE WHEN SUM(Sum_1yProlong_Rub) > 0 THEN SUM(Sum_RateWeighted_1y) / SUM(Sum_1yProlong_Rub) ELSE 0 END AS WeightedRate_1y,
         CASE WHEN SUM(Sum_2yProlong_Rub) > 0 THEN SUM(Sum_RateWeighted_2y) / SUM(Sum_2yProlong_Rub) ELSE 0 END AS WeightedRate_2y,
         CASE WHEN SUM(Sum_3plusProlong_Rub) > 0 THEN SUM(Sum_RateWeighted_3plus) / SUM(Sum_3plusProlong_Rub) ELSE 0 END AS WeightedRate_3plus,
         CASE WHEN SUM(Sum_PreviousBalance) > 0 THEN SUM(Sum_PreviousRateWeighted) / SUM(Sum_PreviousBalance) ELSE 0 END AS WeightedRate_Previous,
         CASE WHEN SUM(Sum_ProlongRub) > 0 THEN SUM(Sum_RateWeighted_Prolong) / SUM(Sum_ProlongRub) ELSE 0 END AS WeightedRate_AllProlong
    FROM OpenAggregated
    GROUP BY CUBE (
         MonthEnd,
         SegmentGrouping,
         CurrencyGrouping,
         TermBucketGrouping
    )
),
-----------------------------
-- B1) Закрытые депозиты (выходы) с расчетом ConvertedRate для закрытых
-----------------------------
ClosedDealsInMonthRates AS (
    SELECT
         M.MonthEnd,
         CAST(dc.CON_ID AS BIGINT) AS CON_ID,
         dc.SEG_NAME,
         dc.CUR,
         dc.DT_OPEN,
         dc.DT_CLOSE,
         dc.BALANCE_RUB,
         dc.RATE,
         DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE_PLAN) AS MATUR,
         conv.NEW_CONVENTION_NAME,
         DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE) AS DaysLived,
         LIQUIDITY.liq.fnc_IntRate(dc.RATE, conv.NEW_CONVENTION_NAME, 'monthly',
                                    DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE_PLAN), 1) AS ConvertedRate
    FROM cteMonths M
    JOIN [LIQUIDITY].[liq].[DepositContract_all] dc
         ON dc.DT_CLOSE >= DATEADD(DAY, 1, EOMONTH(M.MonthEnd, -1))
         AND dc.DT_CLOSE < DATEADD(DAY, 1, M.MonthEnd)
         AND dc.CLI_SUBTYPE = 'INDIV'
         AND dc.PROD_NAME != 'ЭскроУ'
         AND dc.DT_CLOSE_PLAN != '4444-01-01'
         AND DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE) >= 10
    JOIN LIQUIDITY.[liq].man_CONVENTION conv
         ON dc.CONVENTION = conv.CONVENTION_NAME
),
-----------------------------
-- B2) Присоединяем бакеты срочности для закрытых депозитов
-----------------------------
ClosedDealsInMonthBucket AS (
    SELECT
         cdm.MonthEnd,
         cdm.CON_ID,
         cdm.SEG_NAME,
         cdm.CUR,
         cdm.DT_OPEN,
         cdm.DT_CLOSE,
         cdm.BALANCE_RUB,
         cdm.DaysLived,
         cdm.ConvertedRate,
         tg.TERM_GROUP AS TermBucket
    FROM ClosedDealsInMonthRates cdm
    LEFT JOIN [ALM_TEST].[WORK].[man_TermGroup] tg
         ON cdm.DaysLived >= tg.TERM_FROM
         AND cdm.DaysLived <= tg.TERM_TO
),
-----------------------------
-- B3) Агрегируем показатели по закрытым депозитам
-----------------------------
ClosedAggregated AS (
    SELECT
         MonthEnd,
         CASE 
           WHEN SEG_NAME = 'Розничный бизнес' THEN N'Розница'
           WHEN SEG_NAME = 'ДЧБО' THEN N'ЧБО'
           ELSE N'Без сегментации'
         END AS SegmentGrouping,
         CUR AS CurrencyGrouping,
         TermBucket AS TermBucketGrouping,
         COUNT(*) AS ClosedDeals,
         SUM(BALANCE_RUB) AS Summ_ClosedBalanceRub,
         SUM(BALANCE_RUB * ConvertedRate) AS Sum_RateWeighted_Closed
    FROM ClosedDealsInMonthBucket
    GROUP BY CUBE (
         MonthEnd,
         CASE 
           WHEN SEG_NAME = 'Розничный бизнес' THEN N'Розница'
           WHEN SEG_NAME = 'ДЧБО' THEN N'ЧБО'
           ELSE N'Без сегментации'
         END,
         CUR,
         TermBucket
    )
),
-----------------------------
-- B4) Расчет средневзвешенной ставки для закрытых депозитов
-----------------------------
ClosedWeightedRates AS (
    SELECT
         MonthEnd,
         CASE 
           WHEN SEG_NAME = 'Розничный бизнес' THEN N'Розница'
           WHEN SEG_NAME = 'ДЧБО' THEN N'ЧБО'
           ELSE N'Без сегментации'
         END AS SegmentGrouping,
         CUR AS CurrencyGrouping,
         TermBucket AS TermBucketGrouping,
         CASE WHEN SUM(BALANCE_RUB) > 0 THEN SUM(BALANCE_RUB * ConvertedRate) / SUM(BALANCE_RUB)
              ELSE 0 END AS WeightedRate_Closed
    FROM ClosedDealsInMonthBucket
    GROUP BY CUBE (
         MonthEnd,
         CASE 
           WHEN SEG_NAME = 'Розничный бизнес' THEN N'Розница'
           WHEN SEG_NAME = 'ДЧБО' THEN N'ЧБО'
           ELSE N'Без сегментации'
         END,
         CUR,
         TermBucket
    )
)
-----------------------------
-- Финальный SELECT. Объединяем агрегаты по открытым и закрытым депозитам.
-----------------------------
SELECT
    COALESCE(o.MonthEnd, ow.MonthEnd, c.MonthEnd, cw.MonthEnd) AS MonthEnd,
    ISNULL(COALESCE(o.SegmentGrouping, ow.SegmentGrouping, c.SegmentGrouping, cw.SegmentGrouping), N'Все сегменты') AS SegmentGrouping,
    ISNULL(COALESCE(o.CurrencyGrouping, ow.CurrencyGrouping, c.CurrencyGrouping, cw.CurrencyGrouping), N'Все валюты') AS CurrencyGrouping,
    ISNULL(COALESCE(o.TermBucketGrouping, ow.TermBucketGrouping, c.TermBucketGrouping, cw.TermBucketGrouping), N'Все бакеты') AS TermBucketGrouping,
    ISNULL(o.OpenedDeals, 0) AS OpenedDeals,
    ISNULL(o.Summ_BalanceRub, 0) AS Summ_BalanceRub,
    ISNULL(o.Count_Prolong, 0) AS Count_Prolong,
    ISNULL(o.Count_NewNoProlong, 0) AS Count_NewNoProlong,
    ISNULL(o.Count_1yProlong, 0) AS Count_1yProlong,
    ISNULL(o.Count_2yProlong, 0) AS Count_2yProlong,
    ISNULL(o.Count_3plusProlong, 0) AS Count_3plusProlong,
    ISNULL(o.Sum_ProlongRub, 0) AS Sum_ProlongRub,
    ISNULL(o.Sum_NewNoProlong, 0) AS Sum_NewNoProlong,
    ISNULL(o.Sum_1yProlong_Rub, 0) AS Sum_1yProlong_Rub,
    ISNULL(o.Sum_2yProlong_Rub, 0) AS Sum_2yProlong_Rub,
    ISNULL(o.Sum_3plusProlong_Rub, 0) AS Sum_3plusProlong_Rub,
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_ProlongRub / o.Summ_BalanceRub ELSE 0 END AS [Доля_пролонгаций_Rub],
    CASE WHEN ISNULL(o.OpenedDeals, 0) > 0 THEN 1.0 * o.Count_Prolong / o.OpenedDeals ELSE 0 END AS [Доля_пролонгаций_шт],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_1yProlong_Rub / o.Summ_BalanceRub ELSE 0 END AS [Доля_1йПролонгаций_Rub],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_2yProlong_Rub / o.Summ_BalanceRub ELSE 0 END AS [Доля_2йПролонгаций_Rub],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_3plusProlong_Rub / o.Summ_BalanceRub ELSE 0 END AS [Доля_3plusПролонгаций_Rub],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_RateWeighted / o.Summ_BalanceRub ELSE 0 END AS [WeightedRate_All],
    CASE WHEN ISNULL(o.Sum_NewNoProlong, 0) > 0 THEN 1.0 * o.Sum_RateWeighted_NewNoProlong / o.Sum_NewNoProlong ELSE 0 END AS [WeightedRate_NewNoProlong],
    CASE WHEN o.Sum_1yProlong_Rub > 0 THEN o.Sum_RateWeighted_1y / o.Sum_1yProlong_Rub ELSE 0 END AS [WeightedRate_1y],
    CASE WHEN o.Sum_2yProlong_Rub > 0 THEN o.Sum_RateWeighted_2y / o.Sum_2yProlong_Rub ELSE 0 END AS [WeightedRate_2y],
    CASE WHEN o.Sum_3plusProlong_Rub > 0 THEN o.Sum_RateWeighted_3plus / o.Sum_3plusProlong_Rub ELSE 0 END AS [WeightedRate_3plus],
    CASE WHEN ISNULL(o.Sum_PreviousBalance, 0) > 0 THEN 1.0 * o.Sum_PreviousRateWeighted / o.Sum_PreviousBalance ELSE 0 END AS [WeightedRate_Previous],
    CASE WHEN ISNULL(o.Sum_ProlongRub, 0) > 0 THEN 1.0 * o.Sum_RateWeighted_Prolong / o.Sum_ProlongRub ELSE 0 END AS [WeightedRate_AllProlong],
    ISNULL(ow.WeightedRate_All, 0) AS WeightedRate_All_Ow,
    ISNULL(c.ClosedDeals, 0) AS ClosedDeals,
    ISNULL(c.Summ_ClosedBalanceRub, 0) AS Summ_ClosedBalanceRub,
    CASE WHEN ISNULL(c.Summ_ClosedBalanceRub,0) > 0 THEN 1.0 * c.Sum_RateWeighted_Closed / c.Summ_ClosedBalanceRub ELSE 0 END AS [WeightedRate_Closed],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * c.Summ_ClosedBalanceRub / o.Summ_BalanceRub ELSE 0 END AS [Доля_выходов_Rub]
FROM OpenAggregated o
FULL OUTER JOIN OpenWeightedRates ow
    ON o.MonthEnd = ow.MonthEnd
   AND o.SegmentGrouping = ow.SegmentGrouping
   AND o.CurrencyGrouping = ow.CurrencyGrouping
   AND o.TermBucketGrouping = ow.TermBucketGrouping
FULL OUTER JOIN ClosedAggregated c
    ON COALESCE(o.MonthEnd, ow.MonthEnd) = c.MonthEnd
   AND COALESCE(o.SegmentGrouping, ow.SegmentGrouping) = c.SegmentGrouping
   AND COALESCE(o.CurrencyGrouping, ow.CurrencyGrouping) = c.CurrencyGrouping
   AND COALESCE(o.TermBucketGrouping, ow.TermBucketGrouping) = c.TermBucketGrouping
FULL OUTER JOIN ClosedWeightedRates cw
    ON c.MonthEnd = cw.MonthEnd
   AND c.SegmentGrouping = cw.SegmentGrouping
   AND c.CurrencyGrouping = cw.CurrencyGrouping
   AND c.TermBucketGrouping = cw.TermBucketGrouping
ORDER BY
    COALESCE(o.MonthEnd, ow.MonthEnd, c.MonthEnd, cw.MonthEnd);
GO
