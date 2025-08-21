----------------------------------
-- A9) Средневзвешенные ставки (OpenWeightedRates)
----------------------------------
OpenWeightedRates AS (
    SELECT
         MonthEnd,
         SegmentGrouping,
         PROD_NAME,
         CurrencyGrouping,
         TermBucketGrouping,
         BalanceBucketGrouping,
         IS_OPTION,

         -- Ставки
         CASE WHEN SUM(Summ_BalanceRub) > 0
              THEN SUM(Sum_RateWeighted) / SUM(Summ_BalanceRub)
              ELSE 0 END AS WeightedRate_All,

         CASE WHEN SUM(Sum_NewNoProlong) > 0
              THEN SUM(Sum_RateWeighted_NewNoProlong) / SUM(Sum_NewNoProlong)
              ELSE 0 END AS WeightedRate_NewNoProlong,

         CASE WHEN SUM(Sum_1yProlong_Rub) > 0
              THEN SUM(Sum_RateWeighted_1y) / SUM(Sum_1yProlong_Rub)
              ELSE 0 END AS WeightedRate_1y,

         CASE WHEN SUM(Sum_2yProlong_Rub) > 0
              THEN SUM(Sum_RateWeighted_2y) / SUM(Sum_2yProlong_Rub)
              ELSE 0 END AS WeightedRate_2y,

         CASE WHEN SUM(Sum_3yProlong_Rub) > 0
              THEN SUM(Sum_RateWeighted_3y) / SUM(Sum_3yProlong_Rub)
              ELSE 0 END AS WeightedRate_3y,

         CASE WHEN SUM(Sum_4yProlong_Rub) > 0
              THEN SUM(Sum_RateWeighted_4y) / SUM(Sum_4yProlong_Rub)
              ELSE 0 END AS WeightedRate_4y,

         CASE WHEN SUM(Sum_5plusProlong_Rub) > 0
              THEN SUM(Sum_RateWeighted_5plus) / SUM(Sum_5plusProlong_Rub)
              ELSE 0 END AS WeightedRate_5plus,

         CASE WHEN SUM(Sum_ProlongRub) > 0
              THEN SUM(Sum_RateWeighted_Prolong) / SUM(Sum_ProlongRub)
              ELSE 0 END AS WeightedRate_AllProlong,

         CASE WHEN SUM(Sum_PreviousBalance) > 0
              THEN SUM(Sum_PreviousRateWeighted) / SUM(Sum_PreviousBalance)
              ELSE 0 END AS WeightedRate_Previous,

--------------------------------------------------------------------
-- Дисконты (исправлено, проверяем именно *_RubD)
--------------------------------------------------------------------
         CASE WHEN SUM(Summ_BalanceRubD) > 0
              THEN SUM(Sum_Discount) / SUM(Summ_BalanceRubD)
              ELSE 0 END AS WeightedDiscount_All,

         CASE WHEN SUM(Sum_NewNoProlongD) > 0
              THEN SUM(Sum_Discount_NewNoProlong) / SUM(Sum_NewNoProlongD)
              ELSE 0 END AS WeightedDiscount_NewNoProlong,

         CASE WHEN SUM(Sum_1yProlong_RubD) > 0
              THEN SUM(Sum_Discount_1y) / SUM(Sum_1yProlong_RubD)
              ELSE 0 END AS WeightedDiscount_1y,

         CASE WHEN SUM(Sum_2yProlong_RubD) > 0
              THEN SUM(Sum_Discount_2y) / SUM(Sum_2yProlong_RubD)
              ELSE 0 END AS WeightedDiscount_2y,

         CASE WHEN SUM(Sum_3yProlong_RubD) > 0
              THEN SUM(Sum_Discount_3y) / SUM(Sum_3yProlong_RubD)
              ELSE 0 END AS WeightedDiscount_3y,

         CASE WHEN SUM(Sum_4yProlong_RubD) > 0
              THEN SUM(Sum_Discount_4y) / SUM(Sum_4yProlong_RubD)
              ELSE 0 END AS WeightedDiscount_4y,

         CASE WHEN SUM(Sum_5plusProlong_RubD) > 0
              THEN SUM(Sum_Discount_5plus) / SUM(Sum_5plusProlong_RubD)
              ELSE 0 END AS WeightedDiscount_5plus,

         CASE WHEN SUM(Sum_ProlongRubD) > 0
              THEN SUM(Sum_Discount_Prolong) / SUM(Sum_ProlongRubD)
              ELSE 0 END AS WeightedDiscount_AllProlong
    FROM OpenAggregated
    GROUP BY
         MonthEnd, SegmentGrouping, PROD_NAME, CurrencyGrouping,
         TermBucketGrouping, BalanceBucketGrouping, IS_OPTION
)
