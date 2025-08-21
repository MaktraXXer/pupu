меня интересует чтоб по всем открытым депозитам были такими же как в скрипте ниже до этапа создания OpenAggregated_0

по закрытым аналогично чтоб все те поля что в ClosedAggregated_0 возникли

и ты выводил данные исходя из структуры запроса

/*=====================================================================
  Витрина автопролонгаций ФЛ (версия с явными правилами BaseRate)
  ---------------------------------------------------------------------
  • ProlongCount = 0  →  BaseRate = ConvertedRate,  Discount = 0
  • ProlongCount > 0, но база не найдена → BaseRate = NULL, Discount = NULL
=====================================================================*/
USE ALM_TEST;
GO
SET ANSI_NULLS , QUOTED_IDENTIFIER ON;
GO

DECLARE @DayWindow int = 3;               -- ±N дней поиска базы
/*--------------------------------------------------------------------*/
WITH
ISOPT AS (
    SELECT b.CON_ID,
           CASE WHEN ISNULL(b.OptionRate_TRF,0) < 0 THEN 1 ELSE 0 END AS IS_OPTION
    FROM   LIQUIDITY.liq.InterestsRateForDeposit b WITH (NOLOCK)
),
cteMonths AS ( SELECT CONVERT(date,'2025-04-30') AS MonthEnd ),
/*--------------------------------------------------------------------*/
DealsInMonthRates AS (
    SELECT
        M.MonthEnd,
        dc.CON_ID,
        dc.CLI_ID,
        dc.SEG_NAME,
        ISNULL(dc.PROD_NAME,'Без типа')            AS PROD_NAME,
        dc.CUR,
        dc.DT_OPEN,
        dc.DT_CLOSE,
        dc.BALANCE_RUB,
        dc.RATE,
        DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE_PLAN)  AS MATUR,
        conv.NEW_CONVENTION_NAME,
        DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE)       AS DaysLived,
        ISNULL(snap.MonthlyCONV_RATE,
               LIQUIDITY.liq.fnc_IntRate(
                   (dc.RATE+0.0048)/0.9525,
                   conv.NEW_CONVENTION_NAME,
                   'monthly',
                   DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE_PLAN),
                   1) *0.9525 -0.0048)            AS ConvertedRate
    FROM   cteMonths M
    JOIN   LIQUIDITY.liq.DepositContract_all dc WITH (NOLOCK)
           ON dc.DT_OPEN >= DATEADD(DAY,1,EOMONTH(M.MonthEnd,-1))
          AND dc.DT_OPEN <  DATEADD(DAY,1,M.MonthEnd)
          AND dc.CLI_SUBTYPE = 'INDIV'
          AND dc.PROD_NAME  <> 'Эскроу'
          AND dc.CONVENTION <> 'ON_DEMAND'
          AND dc.DT_CLOSE_PLAN <> '4444-01-01'
          AND DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE) > 10
    JOIN   LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
           ON conv.CONVENTION_NAME = dc.CONVENTION
    LEFT  JOIN ALM_TEST.WORK.DepositInterestsRateSnap snap WITH (NOLOCK)
           ON snap.CON_ID = dc.CON_ID
    WHERE  dc.CON_ID NOT IN (SELECT CON_ID
                             FROM LIQUIDITY.liq.man_FloatContracts WITH (NOLOCK))
),
/*--------------------------------------------------------------------*/
DealsInMonthBucket AS (
    SELECT dmr.*,
           tg.TERM_GROUP             AS TermBucket,
           bal.BALANCE_GROUP         AS BalanceBucket,
           CAST(ISNULL(opt.IS_OPTION,0) AS nvarchar) AS IS_OPTION
    FROM   DealsInMonthRates dmr
    LEFT  JOIN ALM_TEST.WORK.man_TermGroup tg  WITH (NOLOCK)
           ON dmr.MATUR BETWEEN tg.TERM_FROM AND tg.TERM_TO
    LEFT  JOIN ALM_TEST.WORK.man_BalanceGroup bal WITH (NOLOCK)
           ON dmr.BALANCE_RUB >= bal.BALANCE_FROM
          AND dmr.BALANCE_RUB <  bal.BALANCE_TO
    LEFT  JOIN ISOPT opt WITH (NOLOCK)
           ON opt.CON_ID = dmr.CON_ID
),
/*--------------------------------------------------------------------*/
RecursiveChain AS (
    SELECT CAST(CON_ID AS bigint) AS StartConId,
           CAST(CON_ID AS bigint) AS CurrentConId,
           0 AS Lvl
    FROM DealsInMonthBucket
    UNION ALL
    SELECT rc.StartConId,
           CAST(p.CON_ID_REL AS bigint),
           rc.Lvl + 1
    FROM RecursiveChain rc
    JOIN ALM.ehd.conrel_prolongations p WITH (NOLOCK)
           ON p.CON_ID = rc.CurrentConId
          AND p.CON_REL_TYPE  = 'PREVIOUS'
    WHERE rc.Lvl < 5
),
ChainDepth AS (
    SELECT StartConId,
           CASE WHEN MAX(Lvl)>5 THEN 5 ELSE MAX(Lvl) END AS ProlongCount
    FROM   RecursiveChain
    GROUP BY StartConId
),
FirstProlong AS (
    SELECT StartConId,
           MIN(CurrentConId) AS Old1
    FROM RecursiveChain
    WHERE Lvl = 1
    GROUP BY StartConId
),
PrevInfo AS (
    SELECT fp.StartConId,
           old.BALANCE_RUB AS PrevBalance,
           LIQUIDITY.liq.fnc_IntRate(
                 old.RATE,
                 conv_prev.NEW_CONVENTION_NAME,
                 'monthly',
                 DATEDIFF(DAY,old.DT_OPEN,old.DT_CLOSE_PLAN),
                 1)                            AS PrevConvertedRate
    FROM FirstProlong fp
    JOIN LIQUIDITY.liq.DepositContract_all old WITH (NOLOCK)
           ON old.CON_ID = fp.Old1
    JOIN LIQUIDITY.liq.man_CONVENTION conv_prev WITH (NOLOCK)
           ON conv_prev.CONVENTION_NAME = old.CONVENTION
),
DealsWithProlong AS (
    SELECT dmb.*,
           ISNULL(cd.ProlongCount,0)    AS ProlongCount,
           ISNULL(pi.PrevBalance,0)     AS PrevBalance,
           ISNULL(pi.PrevConvertedRate,0) AS PrevConvertedRate
    FROM DealsInMonthBucket dmb
    LEFT JOIN ChainDepth cd ON cd.StartConId  = dmb.CON_ID
    LEFT JOIN PrevInfo  pi ON pi.StartConId   = dmb.CON_ID
),
/*--------------------------------------------------------------------*/
/*  Подготовка для поиска базовой ставки                              */
/*--------------------------------------------------------------------*/
BalanceBuckets AS (
    SELECT BALANCE_GROUP,
           BALANCE_FROM,
           BALANCE_TO,
           ROW_NUMBER() OVER(ORDER BY BALANCE_FROM) AS BucketOrder
    FROM ALM_TEST.WORK.man_BalanceGroup
),
BaseCandidates AS (
    SELECT dwp.DT_OPEN,
           dwp.SEG_NAME, dwp.PROD_NAME, dwp.CUR,
           dwp.TermBucket,
           bb.BucketOrder,
           dwp.BalanceBucket,
           dwp.IS_OPTION,
           dwp.NEW_CONVENTION_NAME,
           AVG(dwp.ConvertedRate)       AS BaseRate
    FROM DealsWithProlong dwp
    JOIN BalanceBuckets bb ON bb.BALANCE_GROUP = dwp.BalanceBucket
    WHERE dwp.ProlongCount = 0
    GROUP BY dwp.DT_OPEN, dwp.SEG_NAME, dwp.PROD_NAME, dwp.CUR,
             dwp.TermBucket, bb.BucketOrder, dwp.BalanceBucket,
             dwp.IS_OPTION, dwp.NEW_CONVENTION_NAME
),
DealKeys AS (
    SELECT DISTINCT
           d.DT_OPEN,
           d.SEG_NAME,
           d.PROD_NAME,
           d.CUR,
           d.TermBucket,
           d.IS_OPTION,
           bb.BucketOrder,
           bb.BALANCE_TO            AS BucketUpper,
           d.BalanceBucket,
           d.NEW_CONVENTION_NAME
    FROM DealsWithProlong d
    JOIN BalanceBuckets bb ON bb.BALANCE_GROUP = d.BalanceBucket
),
BestBase AS (
    SELECT dk.*,
           bc.BaseRate,
           ROW_NUMBER() OVER (
                 PARTITION BY dk.DT_OPEN, dk.SEG_NAME, dk.PROD_NAME,
                              dk.CUR,     dk.TermBucket, dk.IS_OPTION,
                              dk.NEW_CONVENTION_NAME,    dk.BalanceBucket
                 ORDER BY
                     CASE WHEN bc.BaseRate IS NULL THEN 1 ELSE 0 END,
                     ABS(DATEDIFF(DAY, bc.DT_OPEN, dk.DT_OPEN)),
                     CASE WHEN dk.BucketUpper<=1500000
                          THEN dk.BucketOrder - ISNULL(bbSrc.BucketOrder,dk.BucketOrder)
                          ELSE ISNULL(bbSrc.BucketOrder,dk.BucketOrder) - dk.BucketOrder
                     END,
                     CASE WHEN bc.DT_OPEN < dk.DT_OPEN THEN 0 ELSE 1 END
           ) AS rn
    FROM DealKeys dk
    LEFT JOIN BaseCandidates bc
           ON bc.SEG_NAME            = dk.SEG_NAME
          AND bc.PROD_NAME           = dk.PROD_NAME
          AND bc.CUR                 = dk.CUR
          AND bc.TermBucket          = dk.TermBucket
          AND bc.IS_OPTION           = dk.IS_OPTION
          AND bc.NEW_CONVENTION_NAME = dk.NEW_CONVENTION_NAME
          AND ABS(DATEDIFF(DAY, bc.DT_OPEN, dk.DT_OPEN)) <= @DayWindow
          AND ( (dk.BucketUpper<=1500000 AND bc.BucketOrder<=dk.BucketOrder)
             OR  (dk.BucketUpper> 1500000 AND bc.BucketOrder>=dk.BucketOrder) )
    LEFT JOIN BalanceBuckets bbSrc ON bbSrc.BALANCE_GROUP = bc.BalanceBucket
),
DailyBaseRate AS (
    SELECT * FROM BestBase WHERE rn = 1         -- ровно одна строка
),
/*--------------------------------------------------------------------*/
/*  Финальная таблица с учётом правил                                 */
/*--------------------------------------------------------------------*/
DealsWithDiscount AS (
    SELECT dwp.*,
           /* --- Правило 1: Prolong = 0 → BaseRate = ConvertedRate --- */
           CASE WHEN dwp.ProlongCount = 0
                THEN dwp.ConvertedRate
                ELSE dbr.BaseRate END           AS BaseRate,
           /* --- Discount --- */
           CASE WHEN dwp.ProlongCount = 0
                THEN 0
                WHEN dbr.BaseRate IS NULL
                THEN NULL
                ELSE dwp.ConvertedRate - dbr.BaseRate END AS Discount
    FROM DealsWithProlong dwp
    LEFT JOIN DailyBaseRate dbr
           ON  dbr.DT_OPEN            = dwp.DT_OPEN
          AND dbr.SEG_NAME           = dwp.SEG_NAME
          AND dbr.PROD_NAME          = dwp.PROD_NAME
          AND dbr.CUR                = dwp.CUR
          AND dbr.TermBucket         = dwp.TermBucket
          AND dbr.BalanceBucket      = dwp.BalanceBucket
          AND dbr.IS_OPTION          = dwp.IS_OPTION
          AND dbr.NEW_CONVENTION_NAME= dwp.NEW_CONVENTION_NAME
),
OpenAggregated_0 AS (
    SELECT 
         MonthEnd,
         CASE 
           WHEN SEG_NAME = 'Розничный бизнес' THEN N'Розница'
           WHEN SEG_NAME = 'ДЧБО' THEN N'ЧБО'
           ELSE N'Без сегментации'
         END AS SegmentGrouping,
		 PROD_NAME,
         CUR AS CurrencyGrouping,
         TermBucket AS TermBucketGrouping,
		 BalanceBucket AS BalanceBucketGrouping,
		 IS_OPTION,

         COUNT(*) AS OpenedDeals,
         SUM(BALANCE_RUB) AS Summ_BalanceRub,

         -- Количество по каждому уровню пролонгации
         SUM(CASE WHEN ProlongCount > 0 THEN 1 ELSE 0 END) AS Count_Prolong,
         SUM(CASE WHEN ProlongCount = 0 THEN 1 ELSE 0 END) AS Count_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN 1 ELSE 0 END) AS Count_1yProlong,
         SUM(CASE WHEN ProlongCount = 2 THEN 1 ELSE 0 END) AS Count_2yProlong,
         SUM(CASE WHEN ProlongCount = 3 THEN 1 ELSE 0 END) AS Count_3yProlong,
         SUM(CASE WHEN ProlongCount = 4 THEN 1 ELSE 0 END) AS Count_4yProlong,
         SUM(CASE WHEN ProlongCount >= 5 THEN 1 ELSE 0 END) AS Count_5plusProlong,

         -- Суммы по уровням пролонгации
         SUM(CASE WHEN ProlongCount > 0 THEN BALANCE_RUB ELSE 0 END) AS Sum_ProlongRub,
         SUM(CASE WHEN ProlongCount = 0 THEN BALANCE_RUB ELSE 0 END) AS Sum_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN BALANCE_RUB ELSE 0 END) AS Sum_1yProlong_Rub,
         SUM(CASE WHEN ProlongCount = 2 THEN BALANCE_RUB ELSE 0 END) AS Sum_2yProlong_Rub,
         SUM(CASE WHEN ProlongCount = 3 THEN BALANCE_RUB ELSE 0 END) AS Sum_3yProlong_Rub,
         SUM(CASE WHEN ProlongCount = 4 THEN BALANCE_RUB ELSE 0 END) AS Sum_4yProlong_Rub,
         SUM(CASE WHEN ProlongCount >= 5 THEN BALANCE_RUB ELSE 0 END) AS Sum_5plusProlong_Rub,
		  -- вспомогательные суммы
		 SUM(CASE WHEN  Discount is not null then BALANCE_RUB ELSE 0 END) AS Summ_BalanceRubD,
         SUM(CASE WHEN  Discount is not null and ProlongCount > 0 THEN BALANCE_RUB ELSE 0 END) AS Sum_ProlongRubD,
         SUM(CASE WHEN  Discount is not null and ProlongCount = 0 THEN BALANCE_RUB ELSE 0 END) AS Sum_NewNoProlongD,
         SUM(CASE WHEN  Discount is not null and ProlongCount = 1 THEN BALANCE_RUB ELSE 0 END) AS Sum_1yProlong_RubD,
         SUM(CASE WHEN Discount is not null and ProlongCount = 2 THEN BALANCE_RUB ELSE 0 END) AS Sum_2yProlong_RubD,
         SUM(CASE WHEN Discount is not null and ProlongCount = 3 THEN BALANCE_RUB ELSE 0 END) AS Sum_3yProlong_RubD,
         SUM(CASE WHEN Discount is not null and ProlongCount = 4 THEN BALANCE_RUB ELSE 0 END) AS Sum_4yProlong_RubD,
         SUM(CASE WHEN Discount is not null and ProlongCount >= 5 THEN BALANCE_RUB ELSE 0 END) AS Sum_5plusProlong_RubD,
         -- Суммы взвешенных ставок
         SUM(BALANCE_RUB * ConvertedRate) AS Sum_RateWeighted,
         SUM(CASE WHEN ProlongCount = 0 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_1y,
         SUM(CASE WHEN ProlongCount = 2 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_2y,
         SUM(CASE WHEN ProlongCount = 3 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_3y,
         SUM(CASE WHEN ProlongCount = 4 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_4y,
         SUM(CASE WHEN ProlongCount >= 5 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_5plus,
         -- Сумма взвешенных ставок по всем пролонгациям + поля для "предыдущего" вклада
         SUM(CASE WHEN ProlongCount > 0 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Sum_RateWeighted_Prolong,

		          -- Суммы взвешенных дисконтов
         SUM(BALANCE_RUB * Discount) AS Sum_Discount,
         SUM(CASE WHEN ProlongCount = 0 THEN BALANCE_RUB * Discount ELSE 0 END) AS Sum_Discount_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN BALANCE_RUB * Discount ELSE 0 END) AS Sum_Discount_1y,
         SUM(CASE WHEN ProlongCount = 2 THEN BALANCE_RUB * Discount ELSE 0 END) AS Sum_Discount_2y,
         SUM(CASE WHEN ProlongCount = 3 THEN BALANCE_RUB * Discount ELSE 0 END) AS Sum_Discount_3y,
         SUM(CASE WHEN ProlongCount = 4 THEN BALANCE_RUB * Discount ELSE 0 END) AS Sum_Discount_4y,
         SUM(CASE WHEN ProlongCount >= 5 THEN BALANCE_RUB * Discount ELSE 0 END) AS Sum_Discount_5plus,
         -- Сумма взвешенных ставок по всем пролонгациям + поля для "предыдущего" вклада
         SUM(CASE WHEN ProlongCount > 0 THEN BALANCE_RUB * Discount ELSE 0 END) AS Sum_Discount_Prolong,



         SUM(CASE WHEN ProlongCount > 0 THEN PrevBalance ELSE 0 END) AS Sum_PreviousBalance,
         SUM(CASE WHEN ProlongCount > 0 THEN PrevBalance * PrevConvertedRate ELSE 0 END) AS Sum_PreviousRateWeighted

    FROM DealsWithDiscount
    GROUP BY CUBE (
         MonthEnd,
         CASE 
           WHEN SEG_NAME = 'Розничный бизнес' THEN N'Розница'
           WHEN SEG_NAME = 'ДЧБО' THEN N'ЧБО'
           ELSE N'Без сегментации'
         END,
         CUR,
		 PROD_NAME,
         TermBucket,
		 BalanceBucket,
		 IS_OPTION
    )
),
----------------------------------
-- A8) Аггрегация по открытым (OpenAggregated)
----------------------------------
OpenAggregated AS (
    SELECT 
		 ISNULL(MonthEnd, '9999-12-31') AS MonthEnd,
		 ISNULL(SegmentGrouping, N'Все сегменты') AS SegmentGrouping,
		 ISNULL(PROD_NAME, N'Все продукты')		 AS PROD_NAME,
		 ISNULL(CurrencyGrouping, N'Все валюты')  AS CurrencyGrouping,
		 ISNULL(TermBucketGrouping, N'Все бакеты')AS TermBucketGrouping,
		 ISNULL(BalanceBucketGrouping, N'Все бакеты')AS BalanceBucketGrouping,
		 ISNULL(IS_OPTION,N'Все признаки опциональности') as IS_OPTION,
		 OpenedDeals,
         Summ_BalanceRub,
		 Count_Prolong,
         Count_NewNoProlong,
         Count_1yProlong,
         Count_2yProlong,
         Count_3yProlong,
         Count_4yProlong,
         Count_5plusProlong,
		 Sum_ProlongRub,
         Sum_NewNoProlong,
         Sum_1yProlong_Rub,
         Sum_2yProlong_Rub,
         Sum_3yProlong_Rub,
         Sum_4yProlong_Rub,
         Sum_5plusProlong_Rub,
		 Summ_BalanceRubD,
		 Sum_ProlongRubD,
         Sum_NewNoProlongD,
         Sum_1yProlong_RubD,
         Sum_2yProlong_RubD,
         Sum_3yProlong_RubD,
         Sum_4yProlong_RubD,
         Sum_5plusProlong_RubD,

		 Sum_RateWeighted,
         Sum_RateWeighted_NewNoProlong,
         Sum_RateWeighted_1y,
         Sum_RateWeighted_2y,
         Sum_RateWeighted_3y,
         Sum_RateWeighted_4y,
         Sum_RateWeighted_5plus,
         Sum_RateWeighted_Prolong,
		 --------------------------
		 Sum_Discount,
		 Sum_Discount_NewNoProlong,
		 Sum_Discount_1y,
		 Sum_Discount_2y,
		 Sum_Discount_3y,
		 Sum_Discount_4y,
		 Sum_Discount_5plus,
		 Sum_Discount_Prolong,

         Sum_PreviousBalance,
         Sum_PreviousRateWeighted
    FROM OpenAggregated_0
    
),
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
         -- Общая
         CASE WHEN SUM(Summ_BalanceRub) > 0
              THEN SUM(Sum_RateWeighted) / SUM(Summ_BalanceRub)
              ELSE 0 END AS WeightedRate_All,

         -- Только новые без пролонгации
         CASE WHEN SUM(Sum_NewNoProlong) > 0
              THEN SUM(Sum_RateWeighted_NewNoProlong) / SUM(Sum_NewNoProlong)
              ELSE 0 END AS WeightedRate_NewNoProlong,

         -- 1-5+ (детальные)
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

         -- Все пролонгированные
         CASE WHEN SUM(Sum_ProlongRub) > 0
              THEN SUM(Sum_RateWeighted_Prolong) / SUM(Sum_ProlongRub)
              ELSE 0 END AS WeightedRate_AllProlong,

         -- Предыдущий вклад (Lvl=1)
         CASE WHEN SUM(Sum_PreviousBalance) > 0
              THEN SUM(Sum_PreviousRateWeighted) / SUM(Sum_PreviousBalance)
              ELSE 0 END AS WeightedRate_Previous,
--------------------------------------------------------------------
			           -- Общая
         CASE WHEN SUM(Summ_BalanceRub) > 0
              THEN SUM(Sum_Discount) / SUM(Summ_BalanceRubD)
              ELSE 0 END AS WeightedDiscount_All,

         -- Только новые без пролонгации
         CASE WHEN SUM(Sum_NewNoProlong) > 0
              THEN SUM(Sum_Discount_NewNoProlong) / SUM(Sum_NewNoProlongD)
              ELSE 0 END AS WeightedDiscount_NewNoProlong,

         -- 1-5+ (детальные)
         CASE WHEN SUM(Sum_1yProlong_Rub) > 0
              THEN SUM(Sum_Discount_1y) / SUM(Sum_1yProlong_RubD)
              ELSE 0 END AS WeightedDiscount_1y,

         CASE WHEN SUM(Sum_2yProlong_Rub) > 0
              THEN SUM(Sum_Discount_2y) / SUM(Sum_2yProlong_RubD)
              ELSE 0 END AS WeightedDiscount_2y,

         CASE WHEN SUM(Sum_3yProlong_Rub) > 0
              THEN SUM(Sum_Discount_3y) / SUM(Sum_3yProlong_RubD)
              ELSE 0 END AS WeightedDiscount_3y,

         CASE WHEN SUM(Sum_4yProlong_Rub) > 0
              THEN SUM(Sum_Discount_4y) / SUM(Sum_4yProlong_RubD)
              ELSE 0 END AS WeightedDiscount_4y,

         CASE WHEN SUM(Sum_5plusProlong_Rub) > 0
              THEN SUM(Sum_Discount_5plus) / SUM(Sum_5plusProlong_RubD)
              ELSE 0 END AS WeightedDiscount_5plus,

         -- Все пролонгированные
         CASE WHEN SUM(Sum_ProlongRub) > 0
              THEN SUM(Sum_Discount_Prolong) / SUM(Sum_ProlongRubD)
              ELSE 0 END AS WeightedDiscount_AllProlong



    FROM OpenAggregated
    GROUP BY
         MonthEnd,
         SegmentGrouping,
		 PROD_NAME,
         CurrencyGrouping,
         TermBucketGrouping,
		 BalanceBucketGrouping,
		 IS_OPTION
),
-------------------------------------------
-- B1) Закрытые депозиты (ClosedDealsInMonthRates)
--     с расчетом ConvertedRate
-------------------------------------------
ClosedDealsInMonthRates AS (
    SELECT
         M.MonthEnd,
         CAST(dc.CON_ID AS BIGINT) AS CON_ID,
         dc.SEG_NAME,
		 ISNULL(dc.PROD_NAME,'Без типа') as PROD_NAME,
         dc.CUR,
         dc.DT_OPEN,
         dc.DT_CLOSE,
         dc.BALANCE_RUB,
         dc.RATE,
         DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE_PLAN) AS MATUR,
         conv.NEW_CONVENTION_NAME,
         DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE) AS DaysLived,
         -- Аналогичная логика конвертации ставки
		   ISNULL(test.[MonthlyCONV_RATE],LIQUIDITY.liq.fnc_IntRate(
             (dc.RATE+0.0048)/0.9525,
             conv.NEW_CONVENTION_NAME,
             'monthly',
             DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE_PLAN),
             1
         )*0.9525 -0.0048) AS ConvertedRate
    FROM cteMonths M
    JOIN [LIQUIDITY].[liq].[DepositContract_all] dc WITH (NOLOCK)
         ON dc.DT_CLOSE >= DATEADD(DAY, 1, EOMONTH(M.MonthEnd, -1))
        AND dc.DT_CLOSE <  DATEADD(DAY, 1, M.MonthEnd)
        AND dc.CLI_SUBTYPE = 'INDIV'
        AND dc.PROD_NAME   != 'Эскроу'
        AND dc.CONVENTION  != 'ON_DEMAND'
        AND dc.DT_CLOSE_PLAN != '4444-01-01'
        AND DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE) > 10
    JOIN LIQUIDITY.[liq].man_CONVENTION conv WITH (NOLOCK)
         ON dc.CONVENTION = conv.CONVENTION_NAME
    left join ALM_TEST.WORK.[DepositInterestsRateSnap] test with (NOLOCK)
		ON  dc.CON_ID =test.CON_ID

	where CAST(dc.CON_ID AS BIGINT) not in (select CAST(CON_ID AS BIGINT) from [LIQUIDITY].[liq].[man_FloatContracts] with (nolock))
),

-------------------------------------------
-- B2) Присоединяем бакеты срочности (ClosedDealsInMonthBucket)
-------------------------------------------
ClosedDealsInMonthBucket AS (
    SELECT
         cdm.MonthEnd,
         cdm.CON_ID,
         cdm.SEG_NAME,
		 cdm.PROD_NAME,
         cdm.CUR,
         cdm.DT_OPEN,
         cdm.DT_CLOSE,
         cdm.BALANCE_RUB,
		 CASE 
			WHEN cdm.CUR ='RUR' and CAST(isnull(opt.[IS_OPTION],0) as nvarchar) ='0'
			THEN 
				CASE
					when cdm.MATUR=cdm.DaysLived then cdm.BALANCE_RUB*(POWER(1+cdm.ConvertedRate/12,cdm.DaysLived/31)-1)
					ELSE  cdm.BALANCE_RUB*(POWER(1+0.01/1200,cdm.DaysLived/31)-1)
			END
		ELSE 0
		END as Int_Balance_RUB,
         cdm.RATE,
         cdm.ConvertedRate,
         cdm.MATUR,
         cdm.DaysLived,
         tg.TERM_GROUP AS TermBucket,
		 bal.BALANCE_GROUP AS BalanceBucket,
		 CAST(isnull(opt.[IS_OPTION],0) as nvarchar) as IS_OPTION
    FROM ClosedDealsInMonthRates cdm
    LEFT JOIN [ALM_TEST].[WORK].[man_TermGroup] tg WITH (NOLOCK)
         ON cdm.MATUR >= tg.TERM_FROM
        AND cdm.MATUR <= tg.TERM_TO
	LEFT JOIN [ALM_TEST].[WORK].[man_BalanceGroup] bal
        ON cdm.BALANCE_RUB >= bal.BALANCE_FROM 
       AND cdm.BALANCE_RUB < bal.BALANCE_TO
	LEFT JOIN ISOPT opt WITH (NOLOCK)
		ON cdm.CON_ID =opt.CON_ID
),

-------------------------------------------
-- B3) (по желанию) Рекурсивная логика для
--     вычисления ProlongCount (0..5) для закрытых
-------------------------------------------
ClosedRecursiveChain AS (
    SELECT
         CAST(dmb.CON_ID AS BIGINT) AS StartConId,
         CAST(dmb.CON_ID AS BIGINT) AS CurrentConId,
         0 AS Lvl
    FROM ClosedDealsInMonthBucket dmb

    UNION ALL

    SELECT
         rc.StartConId,
         CAST(p.CON_ID_REL AS BIGINT) AS CurrentConId,
         rc.Lvl + 1 AS Lvl
    FROM ClosedRecursiveChain rc
    JOIN [ALM].[ehd].[conrel_prolongations] p WITH (NOLOCK)
         ON p.CON_ID = rc.CurrentConId
        AND p.CON_REL_TYPE = 'PREVIOUS'
    WHERE rc.Lvl < 5
),

ClosedChainDepth AS (
    SELECT
         rc.StartConId,
         CASE WHEN MAX(rc.Lvl) > 5 THEN 5 ELSE MAX(rc.Lvl) END AS ProlongCount
    FROM ClosedRecursiveChain rc
    GROUP BY rc.StartConId
),

-------------------------------------------
-- B4) Собираем данные (ClosedDealsWithProlong),
--     но нам НЕ НУЖНО info по предыдущему вкладу
-------------------------------------------
ClosedDealsWithProlong AS (
    SELECT
         dmb.MonthEnd,
         dmb.CON_ID,
         dmb.SEG_NAME,
		 dmb.PROD_NAME,
         dmb.CUR,
         dmb.DT_OPEN,
         dmb.DT_CLOSE,
         dmb.BALANCE_RUB,
		 dmb.Int_Balance_RUB,
         dmb.RATE,
         dmb.MATUR,
         dmb.ConvertedRate,
         dmb.TermBucket,
         dmb.DaysLived,
		 dmb.BalanceBucket,
		 dmb.IS_OPTION,
         ISNULL(cd.ProlongCount, 0) AS ProlongCount
    FROM ClosedDealsInMonthBucket dmb
    LEFT JOIN ClosedChainDepth cd
           ON cd.StartConId = dmb.CON_ID
),

-------------------------------------------
-- B5) Агрегация по закрытым (ClosedAggregated)
--     с учетом ProlongCount (0..5)
-------------------------------------------
ClosedAggregated_0 AS (
    SELECT
         MonthEnd,
         CASE
           WHEN SEG_NAME = 'Розничный бизнес' THEN N'Розница'
           WHEN SEG_NAME = 'ДЧБО' THEN N'ЧБО'
           ELSE N'Без сегментации'
         END AS SegmentGrouping,
		 PROD_NAME,
         CUR AS CurrencyGrouping,
         TermBucket AS TermBucketGrouping,
		 BalanceBucket AS BalanceBucketGrouping,
		 IS_OPTION,
         COUNT(*) AS ClosedDeals,
         SUM(BALANCE_RUB) AS Summ_ClosedBalanceRub,
		 SUM(Int_Balance_RUB) As Summ_ClosedBalanceRub_int,

         -- Аналогично: считаем количество сделок, суммы в разрезе
         SUM(CASE WHEN ProlongCount > 0 THEN 1 ELSE 0 END) AS Closed_Count_Prolong,
         SUM(CASE WHEN ProlongCount = 0 THEN 1 ELSE 0 END) AS Closed_Count_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN 1 ELSE 0 END) AS Closed_Count_1yProlong,
         SUM(CASE WHEN ProlongCount = 2 THEN 1 ELSE 0 END) AS Closed_Count_2yProlong,
         SUM(CASE WHEN ProlongCount = 3 THEN 1 ELSE 0 END) AS Closed_Count_3yProlong,
         SUM(CASE WHEN ProlongCount = 4 THEN 1 ELSE 0 END) AS Closed_Count_4yProlong,
         SUM(CASE WHEN ProlongCount >= 5 THEN 1 ELSE 0 END) AS Closed_Count_5plusProlong,

         SUM(CASE WHEN ProlongCount > 0 THEN BALANCE_RUB ELSE 0 END) AS Closed_Sum_ProlongRub,
         SUM(CASE WHEN ProlongCount = 0 THEN BALANCE_RUB ELSE 0 END) AS Closed_Sum_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN BALANCE_RUB ELSE 0 END) AS Closed_Sum_1yProlong_Rub,
         SUM(CASE WHEN ProlongCount = 2 THEN BALANCE_RUB ELSE 0 END) AS Closed_Sum_2yProlong_Rub,
         SUM(CASE WHEN ProlongCount = 3 THEN BALANCE_RUB ELSE 0 END) AS Closed_Sum_3yProlong_Rub,
         SUM(CASE WHEN ProlongCount = 4 THEN BALANCE_RUB ELSE 0 END) AS Closed_Sum_4yProlong_Rub,
         SUM(CASE WHEN ProlongCount >= 5 THEN BALANCE_RUB ELSE 0 END) AS Closed_Sum_5plusProlong_Rub,

		 -- ПРОЦЕНТЫ
		 SUM(CASE WHEN ProlongCount > 0 THEN Int_Balance_RUB ELSE 0 END) AS Closed_Sum_ProlongRub_int,
         SUM(CASE WHEN ProlongCount = 0 THEN Int_Balance_RUB ELSE 0 END) AS Closed_Sum_NewNoProlong_int,
         SUM(CASE WHEN ProlongCount = 1 THEN Int_Balance_RUB ELSE 0 END) AS Closed_Sum_1yProlong_Rub_int,
         SUM(CASE WHEN ProlongCount = 2 THEN Int_Balance_RUB ELSE 0 END) AS Closed_Sum_2yProlong_Rub_int,
         SUM(CASE WHEN ProlongCount = 3 THEN Int_Balance_RUB ELSE 0 END) AS Closed_Sum_3yProlong_Rub_int,
         SUM(CASE WHEN ProlongCount = 4 THEN Int_Balance_RUB ELSE 0 END) AS Closed_Sum_4yProlong_Rub_int,
         SUM(CASE WHEN ProlongCount >= 5 THEN Int_Balance_RUB ELSE 0 END) AS Closed_Sum_5plusProlong_Rub_int,

         -- Взвешенная сумма
         SUM(BALANCE_RUB * ConvertedRate) AS Closed_Sum_RateWeighted,
		 sum(CASE WHEN ProlongCount > 0 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Closed_Sum_RateWeighted_allProlong,
         SUM(CASE WHEN ProlongCount = 0 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Closed_Sum_RateWeighted_NewNoProlong,
         SUM(CASE WHEN ProlongCount = 1 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Closed_Sum_RateWeighted_1y,
         SUM(CASE WHEN ProlongCount = 2 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Closed_Sum_RateWeighted_2y,
         SUM(CASE WHEN ProlongCount = 3 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Closed_Sum_RateWeighted_3y,
         SUM(CASE WHEN ProlongCount = 4 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Closed_Sum_RateWeighted_4y,
         SUM(CASE WHEN ProlongCount >= 5 THEN BALANCE_RUB * ConvertedRate ELSE 0 END) AS Closed_Sum_RateWeighted_5plus

    FROM ClosedDealsWithProlong
    GROUP BY CUBE (
         MonthEnd,
         CASE
           WHEN SEG_NAME = 'Розничный бизнес' THEN N'Розница'
           WHEN SEG_NAME = 'ДЧБО' THEN N'ЧБО'
           ELSE N'Без сегментации'
         END,
         CUR,
		 PROD_NAME,
         TermBucket,
		 BalanceBucket,
		 IS_OPTION
    )
),
-------------------------------------------
-- B5) Агрегация по закрытым (ClosedAggregated)
--     с учетом ProlongCount (0..5)
-------------------------------------------
ClosedAggregated AS (
    SELECT
         ISNULL(MonthEnd, '9999-12-31') AS MonthEnd,
		 ISNULL(SegmentGrouping, N'Все сегменты') AS SegmentGrouping,
		 ISNULL(PROD_NAME, N'Все продукты')		 AS PROD_NAME,
		 ISNULL(CurrencyGrouping, N'Все валюты')  AS CurrencyGrouping,
		 ISNULL(TermBucketGrouping, N'Все бакеты')AS TermBucketGrouping,
		 ISNULL(BalanceBucketGrouping, N'Все бакеты')AS BalanceBucketGrouping,
		 ISNULL(IS_OPTION,N'Все признаки опциональности') as IS_OPTION,
         ClosedDeals,
         Summ_ClosedBalanceRub,
		 Summ_ClosedBalanceRub_int,
         -- Аналогично: считаем количество сделок, суммы в разрезе
         Closed_Count_Prolong,
         Closed_Count_NewNoProlong,
         Closed_Count_1yProlong,
         Closed_Count_2yProlong,
         Closed_Count_3yProlong,
         Closed_Count_4yProlong,
         Closed_Count_5plusProlong,

         Closed_Sum_ProlongRub,
         Closed_Sum_NewNoProlong,
         Closed_Sum_1yProlong_Rub,
         Closed_Sum_2yProlong_Rub,
         Closed_Sum_3yProlong_Rub,
         Closed_Sum_4yProlong_Rub,
         Closed_Sum_5plusProlong_Rub,

		 Closed_Sum_ProlongRub_int,
         Closed_Sum_NewNoProlong_int,
         Closed_Sum_1yProlong_Rub_int,
         Closed_Sum_2yProlong_Rub_int,
         Closed_Sum_3yProlong_Rub_int,
         Closed_Sum_4yProlong_Rub_int,
         Closed_Sum_5plusProlong_Rub_int,


         -- Взвешенная сумма
         Closed_Sum_RateWeighted,
		 Closed_Sum_RateWeighted_allProlong,
         Closed_Sum_RateWeighted_NewNoProlong,
         Closed_Sum_RateWeighted_1y,
         Closed_Sum_RateWeighted_2y,
         Closed_Sum_RateWeighted_3y,
         Closed_Sum_RateWeighted_4y,
         Closed_Sum_RateWeighted_5plus

    FROM ClosedAggregated_0
),

-------------------------------------------
-- B6) Средневзвешенные ставки (ClosedWeightedRates)
--     по уровню пролонгации (0..5)
-------------------------------------------
ClosedWeightedRates AS (
    SELECT
         MonthEnd,
         SegmentGrouping,
		 PROD_NAME,
         CurrencyGrouping,
         TermBucketGrouping,
		 BalanceBucketGrouping,
		 IS_OPTION,

         -- Общая
         CASE WHEN SUM(Summ_ClosedBalanceRub) > 0
              THEN SUM(Closed_Sum_RateWeighted) / SUM(Summ_ClosedBalanceRub)
              ELSE 0 END AS Closed_WeightedRate_All,

         -- Только новые без пролонгации
         CASE WHEN SUM(Closed_Sum_NewNoProlong) > 0
              THEN SUM(Closed_Sum_RateWeighted_NewNoProlong) / SUM(Closed_Sum_NewNoProlong)
              ELSE 0 END AS Closed_WeightedRate_NewNoProlong,

         -- Детальные 1..5+
         CASE WHEN SUM(Closed_Sum_1yProlong_Rub) > 0
              THEN SUM(Closed_Sum_RateWeighted_1y) / SUM(Closed_Sum_1yProlong_Rub)
              ELSE 0 END AS Closed_WeightedRate_1y,

         CASE WHEN SUM(Closed_Sum_2yProlong_Rub) > 0
              THEN SUM(Closed_Sum_RateWeighted_2y) / SUM(Closed_Sum_2yProlong_Rub)
              ELSE 0 END AS Closed_WeightedRate_2y,

         CASE WHEN SUM(Closed_Sum_3yProlong_Rub) > 0
              THEN SUM(Closed_Sum_RateWeighted_3y) / SUM(Closed_Sum_3yProlong_Rub)
              ELSE 0 END AS Closed_WeightedRate_3y,

         CASE WHEN SUM(Closed_Sum_4yProlong_Rub) > 0
              THEN SUM(Closed_Sum_RateWeighted_4y) / SUM(Closed_Sum_4yProlong_Rub)
              ELSE 0 END AS Closed_WeightedRate_4y,

         CASE WHEN SUM(Closed_Sum_5plusProlong_Rub) > 0
              THEN SUM(Closed_Sum_RateWeighted_5plus) / SUM(Closed_Sum_5plusProlong_Rub)
              ELSE 0 END AS Closed_WeightedRate_5plus,

         -- Все пролонгированные
         CASE WHEN SUM(Closed_Sum_ProlongRub) > 0
              THEN SUM(Closed_Sum_RateWeighted_allProlong) / SUM(Closed_Sum_ProlongRub)
              ELSE 0 END AS Closed_WeightedRate_AllProlong
    FROM ClosedAggregated
    GROUP BY 
         MonthEnd,
         SegmentGrouping,
		 PROD_NAME,
         CurrencyGrouping,
         TermBucketGrouping,
		 BalanceBucketGrouping,
		 is_OPTION
),
------------------------------------------
-- Финальный Шаг: объединяем все CTE
-- (OpenAggregated, OpenWeightedRates,
--  ClosedAggregated, ClosedWeightedRates)
-- через FULL OUTER JOIN, а затем
-- делаем GROUP BY, чтобы "слить" дубли
------------------------------------------
FullJoin AS
(
    SELECT
        --------------------------------
        -- Ключи "Raw*" для дальнейшей группировки
        --------------------------------
        COALESCE(o.MonthEnd, ow.MonthEnd, c.MonthEnd, cw.MonthEnd) AS RawMonthEnd,
        COALESCE(o.SegmentGrouping, ow.SegmentGrouping, c.SegmentGrouping, cw.SegmentGrouping) AS RawSegmentGrouping,
		COALESCE(o.PROD_NAME,ow.PROD_NAME,c.PROD_NAME,cw.PROD_NAME) AS RawPROD_NAME,
        COALESCE(o.CurrencyGrouping, ow.CurrencyGrouping, c.CurrencyGrouping, cw.CurrencyGrouping) AS RawCurrencyGrouping,
        COALESCE(o.TermBucketGrouping, ow.TermBucketGrouping, c.TermBucketGrouping, cw.TermBucketGrouping) AS RawTermBucketGrouping,
		COALESCE(o.BalanceBucketGrouping,ow.BalanceBucketGrouping,c.BalanceBucketGrouping,cw.BalanceBucketGrouping) AS RawBalanceBucketGrouping,
		COALESCE(o.IS_OPTION, ow.IS_OPTION,c.IS_OPTION,cw.IS_OPTION) AS RawIS_OPTION,
        --------------------------------
        -- Поля по ОТКРЫТЫМ ВКЛАДАМ
        --------------------------------
        o.OpenedDeals,
        o.Summ_BalanceRub,

        o.Count_Prolong,
        o.Count_NewNoProlong,
        o.Count_1yProlong,
        o.Count_2yProlong,
        o.Count_3yProlong,
        o.Count_4yProlong,
        o.Count_5plusProlong,

        o.Sum_ProlongRub,
        o.Sum_NewNoProlong,
        o.Sum_1yProlong_Rub,
        o.Sum_2yProlong_Rub,
        o.Sum_3yProlong_Rub,
        o.Sum_4yProlong_Rub,
        o.Sum_5plusProlong_Rub,

        -- Доли открытых (считаем "на лету")
        CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0
             THEN 1.0 * o.Sum_ProlongRub / o.Summ_BalanceRub
             ELSE 0
        END AS Dolya_ProlongRub,

        CASE WHEN ISNULL(o.OpenedDeals, 0) > 0
             THEN 1.0 * o.Count_Prolong / o.OpenedDeals
             ELSE 0
        END AS Dolya_ProlongSht,

        CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0
             THEN 1.0 * o.Sum_1yProlong_Rub / o.Summ_BalanceRub
             ELSE 0
        END AS Dolya_1yProlongRub,

        CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0
             THEN 1.0 * o.Sum_2yProlong_Rub / o.Summ_BalanceRub
             ELSE 0
        END AS Dolya_2yProlongRub,

        CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0
             THEN 1.0 * o.Sum_3yProlong_Rub / o.Summ_BalanceRub
             ELSE 0
        END AS Dolya_3yProlongRub,

        CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0
             THEN 1.0 * o.Sum_4yProlong_Rub / o.Summ_BalanceRub
             ELSE 0
        END AS Dolya_4yProlongRub,

        CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0
             THEN 1.0 * o.Sum_5plusProlong_Rub / o.Summ_BalanceRub
             ELSE 0
        END AS Dolya_5plusProlongRub,

        --------------------------------
        -- Ставки (OpenWeightedRates)
        --------------------------------
        ow.WeightedRate_All,
        ow.WeightedRate_NewNoProlong,
        ow.WeightedRate_AllProlong,
        ow.WeightedRate_Previous,

        ow.WeightedRate_1y,
        ow.WeightedRate_2y,
        ow.WeightedRate_3y,
        ow.WeightedRate_4y,
        ow.WeightedRate_5plus,

		ow.WeightedDiscount_All,
		ow.WeightedDiscount_NewNoProlong,
		ow.WeightedDiscount_AllProlong,
		ow.WeightedDiscount_1y,
		ow.WeightedDiscount_2y,
		ow.WeightedDiscount_3y,
		ow.WeightedDiscount_4y,
		ow.WeightedDiscount_5plus,
        --------------------------------
        -- Поля по ЗАКРЫТЫМ ВКЛАДАМ
        --------------------------------
        c.ClosedDeals,
        c.Summ_ClosedBalanceRub,
		c.Summ_ClosedBalanceRub_int,

        -- "Закрытая" детализация по пролонгациям
        c.Closed_Count_Prolong,
        c.Closed_Count_NewNoProlong,
        c.Closed_Count_1yProlong,
        c.Closed_Count_2yProlong,
        c.Closed_Count_3yProlong,
        c.Closed_Count_4yProlong,
        c.Closed_Count_5plusProlong,

        c.Closed_Sum_ProlongRub,
        c.Closed_Sum_NewNoProlong,
        c.Closed_Sum_1yProlong_Rub,
        c.Closed_Sum_2yProlong_Rub,
        c.Closed_Sum_3yProlong_Rub,
        c.Closed_Sum_4yProlong_Rub,
        c.Closed_Sum_5plusProlong_Rub,

		c.Closed_Sum_ProlongRub_int,
        c.Closed_Sum_NewNoProlong_int,
        c.Closed_Sum_1yProlong_Rub_int,
        c.Closed_Sum_2yProlong_Rub_int,
        c.Closed_Sum_3yProlong_Rub_int,
        c.Closed_Sum_4yProlong_Rub_int,
        c.Closed_Sum_5plusProlong_Rub_int,

        -- Доли закрытых (на лету)
        CASE WHEN ISNULL(c.Summ_ClosedBalanceRub, 0) > 0
             THEN 1.0 * c.Closed_Sum_ProlongRub / c.Summ_ClosedBalanceRub
             ELSE 0
        END AS Closed_Dolya_ProlongRub,

        -- Добавляем доли по 1..5+ для закрытых:
        CASE WHEN ISNULL(c.Summ_ClosedBalanceRub, 0) > 0
             THEN 1.0 * c.Closed_Sum_1yProlong_Rub / c.Summ_ClosedBalanceRub
             ELSE 0
        END AS Closed_Dolya_1yProlongRub,

        CASE WHEN ISNULL(c.Summ_ClosedBalanceRub, 0) > 0
             THEN 1.0 * c.Closed_Sum_2yProlong_Rub / c.Summ_ClosedBalanceRub
             ELSE 0
        END AS Closed_Dolya_2yProlongRub,

        CASE WHEN ISNULL(c.Summ_ClosedBalanceRub, 0) > 0
             THEN 1.0 * c.Closed_Sum_3yProlong_Rub / c.Summ_ClosedBalanceRub
             ELSE 0
        END AS Closed_Dolya_3yProlongRub,

        CASE WHEN ISNULL(c.Summ_ClosedBalanceRub, 0) > 0
             THEN 1.0 * c.Closed_Sum_4yProlong_Rub / c.Summ_ClosedBalanceRub
             ELSE 0
        END AS Closed_Dolya_4yProlongRub,

        CASE WHEN ISNULL(c.Summ_ClosedBalanceRub, 0) > 0
             THEN 1.0 * c.Closed_Sum_5plusProlong_Rub / c.Summ_ClosedBalanceRub
             ELSE 0
        END AS Closed_Dolya_5plusProlongRub,

        --------------------------------
        -- Взвешенная ставка закрытых "в целом"
        --------------------------------
        CASE WHEN ISNULL(c.Summ_ClosedBalanceRub, 0) > 0
             THEN 1.0 * c.Closed_Sum_RateWeighted / c.Summ_ClosedBalanceRub
             ELSE 0
        END AS WeightedRate_Closed_Overall,

        --------------------------------
        -- Ставки (ClosedWeightedRates)
        --------------------------------
        cw.Closed_WeightedRate_All,
        cw.Closed_WeightedRate_NewNoProlong,
        cw.Closed_WeightedRate_1y,
        cw.Closed_WeightedRate_2y,
        cw.Closed_WeightedRate_3y,
        cw.Closed_WeightedRate_4y,
        cw.Closed_WeightedRate_5plus,
        cw.Closed_WeightedRate_AllProlong,

        --------------------------------
        -- Доля выходов
        --------------------------------
        CASE WHEN ISNULL(o.Summ_BalanceRub, 0) > 0
             THEN 1.0 * c.Summ_ClosedBalanceRub / o.Summ_BalanceRub
             ELSE 0
        END AS Dolya_VyhodovRub

    FROM OpenAggregated o
    FULL OUTER JOIN OpenWeightedRates ow
        ON  o.MonthEnd           = ow.MonthEnd
       AND o.SegmentGrouping     = ow.SegmentGrouping
	   AND o.PROD_NAME			 = ow.PROD_NAME
       AND o.CurrencyGrouping    = ow.CurrencyGrouping
       AND o.TermBucketGrouping  = ow.TermBucketGrouping
	   AND o.BalanceBucketGrouping = ow.BalanceBucketGrouping
	   AND o.IS_OPTION =ow.IS_OPTION

    FULL OUTER JOIN ClosedAggregated c
        ON  COALESCE(o.MonthEnd, ow.MonthEnd)          = c.MonthEnd
       AND COALESCE(o.SegmentGrouping, ow.SegmentGrouping) = c.SegmentGrouping
	   AND COALESCE(o.PROD_NAME, ow.PROD_NAME) = c.PROD_NAME
       AND COALESCE(o.CurrencyGrouping, ow.CurrencyGrouping) = c.CurrencyGrouping
       AND COALESCE(o.TermBucketGrouping, ow.TermBucketGrouping) = c.TermBucketGrouping
	   AND COALESCE(o.BalanceBucketGrouping,ow.BalanceBucketGrouping) = c.BalanceBucketGrouping
	   AND COALESCE(o.IS_OPTION,ow.IS_OPTION)=c.IS_OPTION
    FULL OUTER JOIN ClosedWeightedRates cw
        ON  c.MonthEnd           = cw.MonthEnd
       AND c.SegmentGrouping     = cw.SegmentGrouping
	   AND c.PROD_NAME			 = cw.PROD_NAME
       AND c.CurrencyGrouping    = cw.CurrencyGrouping
       AND c.TermBucketGrouping  = cw.TermBucketGrouping
	   AND c.BalanceBucketGrouping = cw.BalanceBucketGrouping
	   AND c.IS_OPTION = cw.IS_OPTION
)
-----
select * from FullJoin;

