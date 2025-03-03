USE ALM_TEST;
GO

CREATE OR ALTER VIEW [WORK].[vw_ProlongationAnalysis_OpenAndClosed_BalanceRub_WithRates]
AS
/*
  Это представление агрегирует данные по депозитам (открытым и закрытым) с 2024 года с подробной сегментацией и расчётом средневзвешенных ставок.

  Основные моменты для открытых депозитов (определяются по DT_OPEN):
  1. Отбираются депозиты, открытые в каждом месяце (по DT_OPEN) и с длительностью ≥ 10 дней.
  2. Для каждого депозита вычисляется MATUR = DATEDIFF(DAY, DT_OPEN, DT_CLOSE_PLAN) и функция 
     LIQUIDITY.liq.fnc_IntRate(…) переводит исходную ставку (RATE) в ежемесячную конвенцию, используя NEW_CONVENTION_NAME,
     полученный из LIQUIDITY.[liq].man_CONVENTION (связь по CONVENTION).
  3. По фактическому количеству дней жизни (DaysLived = DATEDIFF(DAY, DT_OPEN, DT_CLOSE)) определяется бакет срочности (TermBucket)
     через таблицу [man_TermGroup] (поле TERM_GROUP).
  4. Определяется число пролонгаций для каждого депозита. Теперь группировка расширена:
       • Если отсутствует запись – ProlongCount = 0 (новый депозит без пролонгации);
       • Если имеется запись в p1 – ProlongCount = 1;
       • Если p1 и p2 – ProlongCount = 2;
       • Если p1, p2, p3 – ProlongCount = 3;
       • Если p1…p4 – ProlongCount = 4;
       • Если p1…p5 – ProlongCount = 5;
       • И если есть еще – считаем как 5+ (здесь обозначено как 5).
     Для депозитов с ProlongCount > 0 для первого звена (p1, old1) дополнительно рассчитывается PrevConvertedRate и PrevBalance.
  5. В агрегирующем CTE OpenAggregated рассчитываются следующие поля:
       • Количество депозитов (OpenedDeals) и сумма BALANCE_RUB (Summ_BalanceRub);
       • Для каждого разбиения по пролонгациям:
            – Count_NewNoProlong для ProlongCount = 0,
            – Count_1yProlong для ProlongCount = 1,
            – Count_2yProlong для ProlongCount = 2,
            – Count_3yProlong для ProlongCount = 3,
            – Count_4yProlong для ProlongCount = 4,
            – Count_5plusProlong для ProlongCount >= 5.
         Аналогично суммируются BALANCE_RUB по группам:
            – Sum_NewNoProlong, Sum_1yProlong_Rub, Sum_2yProlong_Rub, Sum_3yProlong_Rub, Sum_4yProlong_Rub, Sum_5plusProlong_Rub.
         Также рассчитываются взвешенные суммы ставок:
            – Sum_RateWeighted = SUM(BALANCE_RUB * ConvertedRate) (для всех),
            – Sum_RateWeighted_NewNoProlong для ProlongCount = 0,
            – Sum_RateWeighted_1y для ProlongCount = 1,
            – Sum_RateWeighted_2y для ProlongCount = 2,
            – Sum_RateWeighted_3y для ProlongCount = 3,
            – Sum_RateWeighted_4y для ProlongCount = 4,
            – Sum_RateWeighted_5plus для ProlongCount >= 5.
         Кроме того, рассчитывается:
            – Sum_RateWeighted_Prolong = сумма по депозитам с ProlongCount > 0,
            – Sum_PreviousBalance и Sum_PreviousRateWeighted для предыдущих депозитов.
  6. В итоговом блоке CTE OpenWeightedRates рассчитываются средневзвешенные ставки по открытым:
         • WeightedRate_All = Sum_RateWeighted / Summ_BalanceRub,
         • WeightedRate_NewNoProlong = Sum_RateWeighted_NewNoProlong / Sum_NewNoProlong,
         • WeightedRate_1y = Sum_RateWeighted_1y / Sum_1yProlong_Rub,
         • WeightedRate_2y = Sum_RateWeighted_2y / Sum_2yProlong_Rub,
         • WeightedRate_3y = Sum_RateWeighted_3y / Sum_3yProlong_Rub,
         • WeightedRate_4y = Sum_RateWeighted_4y / Sum_4yProlong_Rub,
         • WeightedRate_5plus = Sum_RateWeighted_5plus / Sum_5plusProlong_Rub,
         • WeightedRate_AllProlong = Sum_RateWeighted_Prolong / (Sum_1yProlong_Rub + Sum_2yProlong_Rub + Sum_3yProlong_Rub + Sum_4yProlong_Rub + Sum_5plusProlong_Rub),
         • WeightedRate_Previous = Sum_PreviousRateWeighted / Sum_PreviousBalance.
  7. Для закрытых депозитов (по DT_CLOSE) аналогичным образом рассчитываются агрегаты ClosedAggregated и средневзвешенная ставка WeightedRate_Closed.
  8. Итоговый SELECT объединяет агрегаты по открытым и закрытым депозитам через FULL OUTER JOIN по ключу
         (MonthEnd, SegmentGrouping, CurrencyGrouping, TermBucketGrouping).
  9. Сегментация выполняется по MonthEnd, SegmentGrouping (с перекодировкой: 'Розничный бизнес'→'Розница', 'ДЧБО'→'ЧБО', иначе 'Без сегментации'), CurrencyGrouping и TermBucketGrouping.
  
  Все агрегаты оборачиваются в ISNULL/COALESCE, чтобы исключить NULL.
  
  Примечание: для простоты примера поля для группировки пролонгаций разделены на:
         - NewNoProlong (ProlongCount = 0),
         - 1y (ProlongCount = 1),
         - 2y (ProlongCount = 2),
         - 3y (ProlongCount = 3),
         - 4y (ProlongCount = 4),
         - 5plus (ProlongCount >= 5).
*/
WITH
-----------------------------
-- 0) Генерация месяцев (MonthEnd) от 2024-01-31 до 2025-02-28
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
    JOIN [LIQUIDITY].[liq].[DepositContract_all] dc WITH (NOLOCK)
         ON dc.DT_OPEN >= DATEADD(DAY, 1, EOMONTH(M.MonthEnd, -1))
        AND dc.DT_OPEN < DATEADD(DAY, 1, M.MonthEnd)
        AND dc.CLI_SUBTYPE = 'INDIV'
        AND dc.PROD_NAME   != 'ЭскроУ'
        AND dc.DT_CLOSE_PLAN != '4444-01-01'
        AND DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE) >= 10
    JOIN LIQUIDITY.[liq].man_CONVENTION conv WITH (NOLOCK)
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
    LEFT JOIN [ALM_TEST].[WORK].[man_TermGroup] tg WITH (NOLOCK)
         ON dmr.DaysLived >= tg.TERM_FROM
        AND dmr.DaysLived <= tg.TERM_TO
),
-----------------------------
-- A3) Определяем число пролонгаций и получаем данные предыдущего депозита
--    Расширяем до 5+ пролонгаций:
--    Если нет p1 – 0, если есть p1 – 1, если p1 и p2 – 2, если p1,p2,p3 – 3, если p1,p2,p3,p4 – 4,
--    если p1,p2,p3,p4,p5 – 5, иначе 5 (5+)
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
           WHEN p4.CON_ID IS NULL OR old4.CON_ID IS NULL THEN 3
           WHEN p5.CON_ID IS NULL OR old5.CON_ID IS NULL THEN 4
           ELSE 5
         END AS ProlongCount,
         CASE WHEN p1.CON_ID IS NOT NULL AND old1.CON_ID IS NOT NULL 
              THEN LIQUIDITY.liq.fnc_IntRate(old1.RATE, conv_prev.NEW_CONVENTION_NAME, 'monthly',
                                              DATEDIFF(DAY, old1.DT_OPEN, old1.DT_CLOSE_PLAN), 1)
              ELSE NULL END AS PrevConvertedRate,
         CASE WHEN p1.CON_ID IS NOT NULL AND old1.CON_ID IS NOT NULL 
              THEN old1.BALANCE_RUB
              ELSE 0 END AS PrevBalance
    FROM DealsInMonthBucket dmb
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p1 WITH (NOLOCK)
         ON p1.CON_ID = dmb.CON_ID
        AND p1.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old1 WITH (NOLOCK)
         ON old1.CON_ID = p1.CON_ID_REL
        AND DATEDIFF(DAY, old1.DT_OPEN, old1.DT_CLOSE) >= 10
    LEFT JOIN LIQUIDITY.[liq].man_CONVENTION conv_prev WITH (NOLOCK)
         ON old1.CONVENTION = conv_prev.CONVENTION_NAME
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p2 WITH (NOLOCK)
         ON p2.CON_ID = old1.CON_ID
        AND p2.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old2 WITH (NOLOCK)
         ON old2.CON_ID = p2.CON_ID_REL
        AND DATEDIFF(DAY, old2.DT_OPEN, old2.DT_CLOSE) >= 10
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p3 WITH (NOLOCK)
         ON p3.CON_ID = old2.CON_ID
        AND p3.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old3 WITH (NOLOCK)
         ON old3.CON_ID = p3.CON_ID_REL
        AND DATEDIFF(DAY, old3.DT_OPEN, old3.DT_CLOSE) >= 10
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p4 WITH (NOLOCK)
         ON p4.CON_ID = old3.CON_ID
        AND p4.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old4 WITH (NOLOCK)
         ON old4.CON_ID = p4.CON_ID_REL
        AND DATEDIFF(DAY, old4.DT_OPEN, old4.DT_CLOSE) >= 10
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p5 WITH (NOLOCK)
         ON p5.CON_ID = old4.CON_ID
        AND p5.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old5 WITH (NOLOCK)
         ON old5.CON_ID = p5.CON_ID_REL
        AND DATEDIFF(DAY, old5.DT_OPEN, old5.DT_CLOSE) >= 10
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
         SUM(CASE WHEN ProlongCount = 3 THEN 1 ELSE 0 END) AS Count_3yProlong,
         SUM(CASE WHEN ProlongCount = 4 THEN 1 ELSE 0 END) AS Count_4yProlong,
         SUM(CASE WHEN ProlongCount >= 5 THEN 1 ELSE 0 END) AS Count_5plusProlong,
         SUM(CASE WHEN ProlongCount > 0 THEN BALANCE_RUB ELSE 0 END) AS Sum_ProlongRub,
         SUM(CASE WHEN ProlongCount = 0 THEN BALANCE_RUB ELSE 0 END) AS Sum_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN BALANCE_RUB ELSE 0 END) AS Sum_1yProlong_Rub,
         SUM(CASE WHEN ProlongCount = 2 THEN BALANCE_RUB ELSE 0 END) AS Sum_2yProlong_Rub,
         SUM(CASE WHEN ProlongCount = 3 THEN BALANCE_RUB ELSE 0 END) AS Sum_3yProlong_Rub,
         SUM(CASE WHEN ProlongCount = 4 THEN BALANCE_RUB ELSE 0 END) AS Sum_4yProlong_Rub,
         SUM(CASE WHEN ProlongCount >= 5 THEN BALANCE_RUB ELSE 0 END) AS Sum_5plusProlong_Rub,
         SUM(BALANCE_RUB * ConvertedRate) AS Sum_RateWeighted,
         SUM(CASE WHEN ProlongCount = 0 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_1y,
         SUM(CASE WHEN ProlongCount = 2 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_2y,
         SUM(CASE WHEN ProlongCount = 3 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_3y,
         SUM(CASE WHEN ProlongCount = 4 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_4y,
         SUM(CASE WHEN ProlongCount >= 5 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_5plus,
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
         CASE WHEN SUM(Sum_3yProlong_Rub) > 0 THEN SUM(Sum_RateWeighted_3y) / SUM(Sum_3yProlong_Rub) ELSE 0 END AS WeightedRate_3y,
         CASE WHEN SUM(Sum_4yProlong_Rub) > 0 THEN SUM(Sum_RateWeighted_4y) / SUM(Sum_4yProlong_Rub) ELSE 0 END AS WeightedRate_4y,
         CASE WHEN SUM(Sum_5plusProlong_Rub) > 0 THEN SUM(Sum_RateWeighted_5plus) / SUM(Sum_5plusProlong_Rub) ELSE 0 END AS WeightedRate_5plus,
         CASE WHEN SUM(Sum_ProlongRub) > 0 
              THEN SUM(Sum_RateWeighted_Prolong) / SUM(Sum_ProlongRub)
              ELSE 0 END AS WeightedRate_AllProlong,
         CASE WHEN SUM(Sum_PreviousBalance) > 0 THEN SUM(Sum_PreviousRateWeighted) / SUM(Sum_PreviousBalance) ELSE 0 END AS WeightedRate_Previous
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
    JOIN [LIQUIDITY].[liq].[DepositContract_all] dc WITH (NOLOCK)
         ON dc.DT_CLOSE >= DATEADD(DAY, 1, EOMONTH(M.MonthEnd, -1))
        AND dc.DT_CLOSE < DATEADD(DAY, 1, M.MonthEnd)
        AND dc.CLI_SUBTYPE = 'INDIV'
        AND dc.PROD_NAME != 'ЭскроУ'
        AND dc.DT_CLOSE_PLAN != '4444-01-01'
        AND DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE) >= 10
    JOIN LIQUIDITY.[liq].man_CONVENTION conv WITH (NOLOCK)
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
    FROM ClosedDealsInMonthRates cdm WITH (NOLOCK)
    LEFT JOIN [ALM_TEST].[WORK].[man_TermGroup] tg WITH (NOLOCK)
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
    ISNULL(o.Count_3yProlong, 0) AS Count_3yProlong,
    ISNULL(o.Count_4yProlong, 0) AS Count_4yProlong,
    ISNULL(o.Count_5plusProlong, 0) AS Count_5plusProlong,
    ISNULL(o.Sum_ProlongRub, 0) AS Sum_ProlongRub,
    ISNULL(o.Sum_NewNoProlong, 0) AS Sum_NewNoProlong,
    ISNULL(o.Sum_1yProlong_Rub, 0) AS Sum_1yProlong_Rub,
    ISNULL(o.Sum_2yProlong_Rub, 0) AS Sum_2yProlong_Rub,
    ISNULL(o.Sum_3yProlong_Rub, 0) AS Sum_3yProlong_Rub,
    ISNULL(o.Sum_4yProlong_Rub, 0) AS Sum_4yProlong_Rub,
    ISNULL(o.Sum_5plusProlong_Rub, 0) AS Sum_5plusProlong_Rub,
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_ProlongRub / o.Summ_BalanceRub ELSE 0 END AS [Доля_пролонгаций_Rub],
    CASE WHEN ISNULL(o.OpenedDeals, 0) > 0 THEN 1.0 * o.Count_Prolong / o.OpenedDeals ELSE 0 END AS [Доля_пролонгаций_шт],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_1yProlong_Rub / o.Summ_BalanceRub ELSE 0 END AS [Доля_1йПролонгаций_Rub],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_2yProlong_Rub / o.Summ_BalanceRub ELSE 0 END AS [Доля_2йПролонгаций_Rub],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_3yProlong_Rub / o.Summ_BalanceRub ELSE 0 END AS [Доля_3йПролонгаций_Rub],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_4yProlong_Rub / o.Summ_BalanceRub ELSE 0 END AS [Доля_4йПролонгаций_Rub],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_5plusProlong_Rub / o.Summ_BalanceRub ELSE 0 END AS [Доля_5plusПролонгаций_Rub],
    CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0 THEN 1.0 * o.Sum_RateWeighted / o.Summ_BalanceRub ELSE 0 END AS [WeightedRate_All],
    CASE WHEN ISNULL(o.Sum_NewNoProlong, 0) > 0 THEN 1.0 * o.Sum_RateWeighted_NewNoProlong / o.Sum_NewNoProlong ELSE 0 END AS [WeightedRate_NewNoProlong],
    CASE WHEN o.Sum_1yProlong_Rub > 0 THEN o.Sum_RateWeighted_1y / o.Sum_1yProlong_Rub ELSE 0 END AS [WeightedRate_1y],
    CASE WHEN o.Sum_2yProlong_Rub > 0 THEN o.Sum_RateWeighted_2y / o.Sum_2yProlong_Rub ELSE 0 END AS [WeightedRate_2y],
    CASE WHEN o.Sum_3yProlong_Rub > 0 THEN o.Sum_RateWeighted_3y / o.Sum_3yProlong_Rub ELSE 0 END AS [WeightedRate_3y],
    CASE WHEN o.Sum_4yProlong_Rub > 0 THEN o.Sum_RateWeighted_4y / o.Sum_4yProlong_Rub ELSE 0 END AS [WeightedRate_4y],
    CASE WHEN o.Sum_5plusProlong_Rub > 0 THEN o.Sum_RateWeighted_5plus / o.Sum_5plusProlong_Rub ELSE 0 END AS [WeightedRate_5plus],
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
GROUP BY COALESCE(o.MonthEnd, ow.MonthEnd, c.MonthEnd, cw.MonthEnd),
         ISNULL(COALESCE(o.SegmentGrouping, ow.SegmentGrouping, c.SegmentGrouping, cw.SegmentGrouping), N'Все сегменты'),
         ISNULL(COALESCE(o.CurrencyGrouping, ow.CurrencyGrouping, c.CurrencyGrouping, cw.CurrencyGrouping), N'Все валюты'),
         ISNULL(COALESCE(o.TermBucketGrouping, ow.TermBucketGrouping, c.TermBucketGrouping, cw.TermBucketGrouping), N'Все бакеты')
ORDER BY COALESCE(o.MonthEnd, ow.MonthEnd, c.MonthEnd, cw.MonthEnd);
GO
