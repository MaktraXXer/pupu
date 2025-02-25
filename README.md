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
    CASE WHEN ISNULL(o.OpenedDeals, 0) > 0 THEN 1.0 * o.Count_1yProlong / o.OpenedDeals ELSE 0 END AS [Доля_1йПролонгаций_шт],
    CASE WHEN ISNULL(o.OpenedDeals, 0) > 0 THEN 1.0 * o.Count_2yProlong / o.OpenedDeals ELSE 0 END AS [Доля_2йПролонгаций_шт],
    CASE WHEN ISNULL(o.OpenedDeals, 0) > 0 THEN 1.0 * o.Count_3plusProlong / o.OpenedDeals ELSE 0 END AS [Доля_3plusПролонгаций_шт],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_RateWeighted / o.Summ_BalanceRub ELSE 0 END AS [WeightedRate_All],
    CASE WHEN ISNULL(o.Sum_NewNoProlong, 0) > 0 THEN 1.0 * o.Sum_RateWeighted_NewNoProlong / o.Sum_NewNoProlong ELSE 0 END AS [WeightedRate_NewNoProlong],
    CASE WHEN o.Sum_1yProlong_Rub > 0 THEN o.Sum_RateWeighted_1y / o.Sum_1yProlong_Rub ELSE 0 END AS [WeightedRate_1y],
    CASE WHEN o.Sum_2yProlong_Rub > 0 THEN o.Sum_RateWeighted_2y / o.Sum_2yProlong_Rub ELSE 0 END AS [WeightedRate_2y],
    CASE WHEN o.Sum_3plusProlong_Rub > 0 THEN o.Sum_RateWeighted_3plus / o.Sum_3plusProlong_Rub ELSE 0 END AS [WeightedRate_3plus],
    CASE WHEN ISNULL(o.Sum_PreviousBalance, 0) > 0 THEN 1.0 * o.Sum_PreviousRateWeighted / o.Sum_PreviousBalance ELSE 0 END AS [WeightedRate_Previous],
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
