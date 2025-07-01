/****** Script for SelectTopNRows command from SSMS  ******/
SELECT  [MonthEnd]
      ,[SegmentGrouping]
      ,[PROD_NAME]
      ,[CurrencyGrouping]
      ,[TermBucketGrouping]
      ,[BalanceBucketGrouping]
      ,[IS_OPTION]
      ,[OpenedDeals]
      ,[Opened_Summ_BalanceRub]
      ,[Opened_Count_Prolong]
      ,[Opened_Count_NewNoProlong]
      ,[Opened_Count_1yProlong]
      ,[Opened_Count_2yProlong]
      ,[Opened_Count_3yProlong]
      ,[Opened_Count_4yProlong]
      ,[Opened_Count_5plusProlong]
      ,[Opened_Sum_ProlongRub]
      ,[Opened_Sum_NewNoProlong]
      ,[Opened_Sum_1yProlong_Rub]
      ,[Opened_Sum_2yProlong_Rub]
      ,[Opened_Sum_3yProlong_Rub]
      ,[Opened_Sum_4yProlong_Rub]
      ,[Opened_Sum_5plusProlong_Rub]
   --   ,[Opened_Dolya_ProlongRub]
    --  ,[Opened_Dolya_ProlongSht]
    --  ,[Opened_Dolya_1yProlongRub]
    --  ,[Opened_Dolya_2yProlongRub]
    --  ,[Opened_Dolya_3yProlongRub]
    --  ,[Opened_Dolya_4yProlongRub]
    --  ,[Opened_Dolya_5plusProlongRub]
      ,[Opened_WeightedRate_All]
      ,[Opened_WeightedRate_NewNoProlong]
      ,[Opened_WeightedRate_AllProlong]
      ,[Opened_WeightedRate_Previous]
      ,[Opened_WeightedRate_1y]
      ,[Opened_WeightedRate_2y]
      ,[Opened_WeightedRate_3y]
      ,[Opened_WeightedRate_4y]
      ,[Opened_WeightedRate_5plus]
      ,[ClosedDeals]
      ,[Summ_ClosedBalanceRub]
      ,[Summ_ClosedBalanceRub_int]
      ,[Closed_Count_Prolong]
      ,[Closed_Count_NewNoProlong]
      ,[Closed_Count_1yProlong]
      ,[Closed_Count_2yProlong]
      ,[Closed_Count_3yProlong]
      ,[Closed_Count_4yProlong]
      ,[Closed_Count_5plusProlong]
      ,[Closed_Sum_ProlongRub]
      ,[Closed_Sum_NewNoProlong]
      ,[Closed_Sum_1yProlong_Rub]
      ,[Closed_Sum_2yProlong_Rub]
      ,[Closed_Sum_3yProlong_Rub]
      ,[Closed_Sum_4yProlong_Rub]
      ,[Closed_Sum_5plusProlong_Rub]
      ,[Closed_Sum_ProlongRub_int]
      ,[Closed_Sum_NewNoProlong_int]
      ,[Closed_Sum_1yProlong_Rub_int]
      ,[Closed_Sum_2yProlong_Rub_int]
      ,[Closed_Sum_3yProlong_Rub_int]
      ,[Closed_Sum_4yProlong_Rub_int]
      ,[Closed_Sum_5plusProlong_Rub_int]
     -- ,[Closed_Dolya_ProlongRub]
    --  ,[Closed_Dolya_1yProlongRub]
    --  ,[Closed_Dolya_2yProlongRub]
   --   ,[Closed_Dolya_3yProlongRub]
    --  ,[Closed_Dolya_4yProlongRub]
    --  ,[Closed_Dolya_5plusProlongRub]
      ,[WeightedRate_Closed_Overall]
      ,[Closed_WeightedRate_All]
      ,[Closed_WeightedRate_NewNoProlong]
      ,[Closed_WeightedRate_1y]
      ,[Closed_WeightedRate_2y]
      ,[Closed_WeightedRate_3y]
      ,[Closed_WeightedRate_4y]
      ,[Closed_WeightedRate_5plus]
      ,[Closed_WeightedRate_AllProlong]
  --    ,[Dolya_VyhodovRub]
  FROM [ALM_TEST].[WORK].[prolongationAnalysisResult_ISOPTION]

следовательно -в экселе у меня такие поля



  вот код как я создал такую витрину

  USE [ALM_TEST]
GO

/****** Object:  View [WORK].[vw_prolongationAnalysis]    Script Date: 11.03.2025 8:38:39 ******/
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

/* DELETE FROM  [WORK].[prolongationAnalysisResult_ISOPTION]
WHERE MonthEnd >='2025-01-31'; */
WITH
ISOPT AS(
SELECT 
        b.[CON_ID],
		CASE 
        WHEN isnull(b.[OptionRate_TRF], 0) <0 THEN 1
       ELSE 0 -- Если продукт не найден в таблице маппинга
    END AS [IS_OPTION]
    FROM [LIQUIDITY].[liq].[InterestsRateForDeposit] b with (nolock)
),
----------------------------------
-- 0) Генерация месяцев (MonthEnd)
----------------------------------
cteMonths AS (
    SELECT CONVERT(date, '2025-03-31') AS MonthEnd
    UNION ALL
    SELECT EOMONTH(DATEADD(MONTH, 1, MonthEnd))
    FROM cteMonths
    WHERE EOMONTH(DATEADD(MONTH, 1, MonthEnd)) <= '2025-04-30'
),

----------------------------------
-- A1) Открытые депозиты (DealsInMonthRates)
----------------------------------
DealsInMonthRates AS (
    SELECT
         M.MonthEnd,
         CAST(dc.CON_ID AS BIGINT) AS CON_ID,
		 dc.CLI_ID,
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
         -- Переводим ставку в ежемесячную конвенцию
         LIQUIDITY.liq.fnc_IntRate(
             dc.RATE,
             conv.NEW_CONVENTION_NAME,
             'monthly',
             DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE_PLAN),
             1
         ) AS ConvertedRate
    FROM cteMonths M
    JOIN [LIQUIDITY].[liq].[DepositContract_all] dc WITH (NOLOCK)
         ON dc.DT_OPEN >= DATEADD(DAY, 1, EOMONTH(M.MonthEnd, -1))
        AND dc.DT_OPEN <  DATEADD(DAY, 1, M.MonthEnd)
        AND dc.CLI_SUBTYPE = 'INDIV'
        AND dc.PROD_NAME   != 'Эскроу'
		AND dc.convention != 'ON_DEMAND'
        AND dc.DT_CLOSE_PLAN != '4444-01-01'
        AND DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE) > 10
    JOIN LIQUIDITY.[liq].man_CONVENTION conv WITH (NOLOCK)
         ON dc.CONVENTION = conv.CONVENTION_NAME
	where CAST(dc.CON_ID AS BIGINT)  not in (select CAST(CON_ID AS BIGINT) from [LIQUIDITY].[liq].[man_FloatContracts] with (nolock))
),

----------------------------------
-- A2) Присоединяем бакеты срочности (DealsInMonthBucket)
----------------------------------
DealsInMonthBucket AS (
    SELECT
         dmr.MonthEnd,
         dmr.CON_ID,
		 dmr.CLI_ID,
         dmr.SEG_NAME,
		 dmr.PROD_NAME,
         dmr.CUR,
         dmr.DT_OPEN,
         dmr.DT_CLOSE,
         dmr.BALANCE_RUB,
         dmr.RATE,
         dmr.ConvertedRate,
         dmr.MATUR,
         dmr.DaysLived,
         tg.TERM_GROUP AS TermBucket,
		 bal.BALANCE_GROUP AS BalanceBucket,
		 CAST(isnull(opt.[IS_OPTION],0) as nvarchar) as IS_OPTION
    FROM DealsInMonthRates dmr
    LEFT JOIN [ALM_TEST].[WORK].[man_TermGroup] tg WITH (NOLOCK)
         ON dmr.MATUR >= tg.TERM_FROM
        AND dmr.MATUR <= tg.TERM_TO
	LEFT JOIN [ALM_TEST].[WORK].[man_BalanceGroup] bal
        ON dmr.BALANCE_RUB >= bal.BALANCE_FROM 
       AND dmr.BALANCE_RUB < bal.BALANCE_TO
	LEFT JOIN ISOPT opt WITH (NOLOCK)
		ON dmr.CON_ID =opt.CON_ID
),

----------------------------------
-- A3) (рекурсивный CTE) 
--    Можно использовать для глубины 5 (5+).
----------------------------------
RecursiveChain AS (
    SELECT
         CAST(dmb.CON_ID AS BIGINT)  AS StartConId,
         CAST(dmb.CON_ID AS BIGINT) AS CurrentConId,
         0           AS Lvl
    FROM DealsInMonthBucket dmb

    UNION ALL

    SELECT
         rc.StartConId,
         CAST(p.CON_ID_REL AS BIGINT) AS CurrentConId,
         rc.Lvl + 1   AS Lvl
    FROM RecursiveChain rc
    JOIN [ALM].[ehd].[conrel_prolongations] p WITH (NOLOCK)
         ON p.CON_ID = rc.CurrentConId
        AND p.CON_REL_TYPE = 'PREVIOUS'
    WHERE rc.Lvl < 5
),

----------------------------------
-- A4) Получаем ProlongCount (0..5)
----------------------------------
ChainDepth AS (
    SELECT
         rc.StartConId,
         CASE WHEN MAX(rc.Lvl) > 5 THEN 5 ELSE MAX(rc.Lvl) END AS ProlongCount
    FROM RecursiveChain rc
    GROUP BY rc.StartConId
),

----------------------------------
-- A5) Находим (Lvl=1) для вычисления PrevBalance, PrevConvertedRate
----------------------------------
FirstProlongation AS (
    SELECT
         r1.StartConId,
         r1.CurrentConId AS Old1_ConId
    FROM RecursiveChain r1
    WHERE r1.Lvl = 1
),
----------------------------------
-- A6) Берём поля предыдущего вклада (Balance, RATE->PrevConvertedRate)
----------------------------------
PreviousDepositInfo AS (
    SELECT
         fp.StartConId,
         old1.CON_ID      AS Old1_ConId,
         old1.BALANCE_RUB AS PrevBalance,
         LIQUIDITY.liq.fnc_IntRate(
             old1.RATE,
             conv_prev.NEW_CONVENTION_NAME,
             'monthly',
             DATEDIFF(DAY, old1.DT_OPEN, old1.DT_CLOSE_PLAN),
             1
         ) AS PrevConvertedRate
    FROM FirstProlongation fp
    JOIN [LIQUIDITY].[liq].[DepositContract_all] old1 WITH (NOLOCK)
         ON old1.CON_ID = fp.Old1_ConId
    JOIN LIQUIDITY.[liq].man_CONVENTION conv_prev WITH (NOLOCK)
         ON old1.CONVENTION = conv_prev.CONVENTION_NAME
),

----------------------------------
-- A7) Собираем данные (DealsWithProlong)
----------------------------------
DealsWithProlong AS (
    SELECT
         dmb.MonthEnd,
         dmb.CON_ID,
		 dmb.CLI_ID,
         dmb.SEG_NAME,
		 dmb.PROD_NAME,
         dmb.CUR,
         dmb.DT_OPEN,
         dmb.DT_CLOSE,
         dmb.BALANCE_RUB,
         dmb.RATE,
         dmb.MATUR,
         dmb.ConvertedRate,
         dmb.TermBucket,
		 dmb.BalanceBucket,
		 dmb.IS_OPTION,
         dmb.DaysLived,
         ISNULL(cd.ProlongCount, 0) AS ProlongCount,
         ISNULL(pdi.PrevBalance, 0) AS PrevBalance,
         ISNULL(pdi.PrevConvertedRate, 0) AS PrevConvertedRate
    FROM DealsInMonthBucket dmb
    LEFT JOIN ChainDepth cd
           ON cd.StartConId = dmb.CON_ID
    LEFT JOIN PreviousDepositInfo pdi
           ON pdi.StartConId = dmb.CON_ID
),
----------------------------------
-- A8) Аггрегация по открытым (OpenAggregated)
----------------------------------
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
		 Sum_RateWeighted,
         Sum_RateWeighted_NewNoProlong,
         Sum_RateWeighted_1y,
         Sum_RateWeighted_2y,
         Sum_RateWeighted_3y,
         Sum_RateWeighted_4y,
         Sum_RateWeighted_5plus,
         Sum_RateWeighted_Prolong,
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
              ELSE 0 END AS WeightedRate_Previous
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
         LIQUIDITY.liq.fnc_IntRate(
             dc.RATE,
             conv.NEW_CONVENTION_NAME,
             'monthly',
             DATEDIFF(DAY, dc.DT_OPEN, dc.DT_CLOSE_PLAN),
             1
         ) AS ConvertedRate
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

------------------------------------------
INSERT INTO [WORK].[prolongationAnalysisResult_ISOPTION]
SELECT 
    -- Заменяем NULL на "Все ...":
    RawMonthEnd AS MonthEnd,
    RawSegmentGrouping AS SegmentGrouping,
	RawPROD_NAME	 AS PROD_NAME,
    FullJoin.RawCurrencyGrouping  AS CurrencyGrouping,
    FullJoin.RawTermBucketGrouping AS TermBucketGrouping,
	FullJoin.RawBalanceBucketGrouping as BalanceBucketGrouping,
	FullJoin.RawIS_OPTION as IS_OPTION,
    OpenedDeals                  AS OpenedDeals,
    Summ_BalanceRub              AS Opened_Summ_BalanceRub,
    Count_Prolong               AS Opened_Count_Prolong,
    Count_NewNoProlong           AS Opened_Count_NewNoProlong,
    Count_1yProlong              AS Opened_Count_1yProlong,
    Count_2yProlong              AS Opened_Count_2yProlong,
    Count_3yProlong              AS Opened_Count_3yProlong,
    Count_4yProlong              AS Opened_Count_4yProlong,
    Count_5plusProlong           AS Opened_Count_5plusProlong,

    Sum_ProlongRub               AS Opened_Sum_ProlongRub,
    Sum_NewNoProlong             AS Opened_Sum_NewNoProlong,
    Sum_1yProlong_Rub            AS Opened_Sum_1yProlong_Rub,
    Sum_2yProlong_Rub            AS Opened_Sum_2yProlong_Rub,
    Sum_3yProlong_Rub            AS Opened_Sum_3yProlong_Rub,
    Sum_4yProlong_Rub            AS Opened_Sum_4yProlong_Rub,
    Sum_5plusProlong_Rub         AS Opened_Sum_5plusProlong_Rub,

    Dolya_ProlongRub            AS Opened_Dolya_ProlongRub,
    Dolya_ProlongSht             AS Opened_Dolya_ProlongSht,
    Dolya_1yProlongRub           AS Opened_Dolya_1yProlongRub,
    Dolya_2yProlongRub           AS Opened_Dolya_2yProlongRub,
    Dolya_3yProlongRub           AS Opened_Dolya_3yProlongRub,
    Dolya_4yProlongRub           AS Opened_Dolya_4yProlongRub,
    Dolya_5plusProlongRub        AS Dolya_5plusProlongRub,

    -- Ставки открытых
    WeightedRate_All             AS Opened_WeightedRate_All,
    WeightedRate_NewNoProlong    AS Opened_WeightedRate_NewNoProlong,
    WeightedRate_AllProlong      AS Opened_WeightedRate_AllProlong,
    WeightedRate_Previous        AS Opened_WeightedRate_Previous,

    WeightedRate_1y              AS Opened_WeightedRate_1y,
    WeightedRate_2y              AS Opened_WeightedRate_2y,
    WeightedRate_3y              AS Opened_WeightedRate_3y,
    WeightedRate_4y              AS Opened_WeightedRate_4y,
    WeightedRate_5plus           AS Opened_WeightedRate_5plus,

    -----------------------------------
    -- Закрытые
    -----------------------------------
    ClosedDeals                       AS ClosedDeals,
    Summ_ClosedBalanceRub             AS Summ_ClosedBalanceRub,
	Summ_ClosedBalanceRub_int         AS Summ_ClosedBalanceRub_int,

    Closed_Count_Prolong              AS Closed_Count_Prolong,
    Closed_Count_NewNoProlong         AS Closed_Count_NewNoProlong,
    Closed_Count_1yProlong            AS Closed_Count_1yProlong,
    Closed_Count_2yProlong            AS Closed_Count_2yProlong,
    Closed_Count_3yProlong            AS Closed_Count_3yProlong,
    Closed_Count_4yProlong            AS Closed_Count_4yProlong,
    Closed_Count_5plusProlong         AS Closed_Count_5plusProlong,

    Closed_Sum_ProlongRub             AS Closed_Sum_ProlongRub,
    Closed_Sum_NewNoProlong           AS Closed_Sum_NewNoProlong,
    Closed_Sum_1yProlong_Rub          AS Closed_Sum_1yProlong_Rub,
    Closed_Sum_2yProlong_Rub          AS Closed_Sum_2yProlong_Rub,
    Closed_Sum_3yProlong_Rub          AS Closed_Sum_3yProlong_Rub,
    Closed_Sum_4yProlong_Rub          AS Closed_Sum_4yProlong_Rub,
    Closed_Sum_5plusProlong_Rub       AS Closed_Sum_5plusProlong_Rub,

	Closed_Sum_ProlongRub_int             AS Closed_Sum_ProlongRub_int,
    Closed_Sum_NewNoProlong_int           AS Closed_Sum_NewNoProlong_int,
    Closed_Sum_1yProlong_Rub_int          AS Closed_Sum_1yProlong_Rub_int,
    Closed_Sum_2yProlong_Rub_int          AS Closed_Sum_2yProlong_Rub_int,
    Closed_Sum_3yProlong_Rub_int          AS Closed_Sum_3yProlong_Rub_int,
    Closed_Sum_4yProlong_Rub_int          AS Closed_Sum_4yProlong_Rub_int,
    Closed_Sum_5plusProlong_Rub_int       AS Closed_Sum_5plusProlong_Rub_int,

    Closed_Dolya_ProlongRub           AS Closed_Dolya_ProlongRub,
    Closed_Dolya_1yProlongRub         AS Closed_Dolya_1yProlongRub,
    Closed_Dolya_2yProlongRub         AS Closed_Dolya_2yProlongRub,
    Closed_Dolya_3yProlongRub         AS Closed_Dolya_3yProlongRub,
    Closed_Dolya_4yProlongRub         AS Closed_Dolya_4yProlongRub,
    Closed_Dolya_5plusProlongRub      AS Closed_Dolya_5plusProlongRub,

    WeightedRate_Closed_Overall       AS WeightedRate_Closed_Overall,

    Closed_WeightedRate_All           AS Closed_WeightedRate_All,
    Closed_WeightedRate_NewNoProlong  AS Closed_WeightedRate_NewNoProlong,
    Closed_WeightedRate_1y            AS Closed_WeightedRate_1y,
    Closed_WeightedRate_2y            AS Closed_WeightedRate_2y,
    Closed_WeightedRate_3y            AS Closed_WeightedRate_3y,
    Closed_WeightedRate_4y            AS Closed_WeightedRate_4y,
    Closed_WeightedRate_5plus         AS Closed_WeightedRate_5plus,
    Closed_WeightedRate_AllProlong    AS Closed_WeightedRate_AllProlong,

    Dolya_VyhodovRub                  AS Dolya_VyhodovRub
FROM FullJoin
/*GROUP BY
    RawMonthEnd,
    RawSegmentGrouping,
	RawPROD_NAME,
    RawCurrencyGrouping,
    RawTermBucketGrouping
	*/
ORDER BY
    RawMonthEnd;

GO




2	Общая пролонгация	=ЕСЛИОШИБКА(Opened_Sum_ProlongRub/(Summ_ClosedBalanceRub+Summ_ClosedBalanceRub_int);0)
4	1-ая автопролонгация	=ЕСЛИОШИБКА(Opened_Sum_1yProlong_Rub/(Closed_Sum_NewNoProlong+Closed_Sum_NewNoProlong_int);0)

6	2-ая автопролонгация 	=ЕСЛИОШИБКА(Opened_Sum_2yProlong_Rub/(Closed_Sum_1yProlong_Rub+Closed_Sum_1yProlong_Rub_int);0)


8	3-ая автопролонгация 	=ЕСЛИОШИБКА(Opened_Sum_3yProlong_Rub/(Closed_Sum_2yProlong_Rub+Closed_Sum_2yProlong_Rub_int);0)


твоя задача детально и хорошо запомнить структуру данных таблиц
