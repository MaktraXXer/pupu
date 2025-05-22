/\*\*\*\*\*\* Script for SelectTopNRows command from SSMS  \*\*\*\*\*\*/
SELECT  \[MonthEnd]
,\[SegmentGrouping]
,\[PROD\_NAME]
,\[CurrencyGrouping]
,\[TermBucketGrouping]
,\[BalanceBucketGrouping]
,\[IS\_OPTION]
,\[OpenedDeals]
,\[Opened\_Summ\_BalanceRub]
,\[Opened\_Count\_Prolong]
,\[Opened\_Count\_NewNoProlong]
,\[Opened\_Count\_1yProlong]
,\[Opened\_Count\_2yProlong]
,\[Opened\_Count\_3yProlong]
,\[Opened\_Count\_4yProlong]
,\[Opened\_Count\_5plusProlong]
,\[Opened\_Sum\_ProlongRub]
,\[Opened\_Sum\_NewNoProlong]
,\[Opened\_Sum\_1yProlong\_Rub]
,\[Opened\_Sum\_2yProlong\_Rub]
,\[Opened\_Sum\_3yProlong\_Rub]
,\[Opened\_Sum\_4yProlong\_Rub]
,\[Opened\_Sum\_5plusProlong\_Rub]
\--   ,\[Opened\_Dolya\_ProlongRub]
\--  ,\[Opened\_Dolya\_ProlongSht]
\--  ,\[Opened\_Dolya\_1yProlongRub]
\--  ,\[Opened\_Dolya\_2yProlongRub]
\--  ,\[Opened\_Dolya\_3yProlongRub]
\--  ,\[Opened\_Dolya\_4yProlongRub]
\--  ,\[Opened\_Dolya\_5plusProlongRub]
,\[Opened\_WeightedRate\_All]
,\[Opened\_WeightedRate\_NewNoProlong]
,\[Opened\_WeightedRate\_AllProlong]
,\[Opened\_WeightedRate\_Previous]
,\[Opened\_WeightedRate\_1y]
,\[Opened\_WeightedRate\_2y]
,\[Opened\_WeightedRate\_3y]
,\[Opened\_WeightedRate\_4y]
,\[Opened\_WeightedRate\_5plus]
,\[ClosedDeals]
,\[Summ\_ClosedBalanceRub]
,\[Summ\_ClosedBalanceRub\_int]
,\[Closed\_Count\_Prolong]
,\[Closed\_Count\_NewNoProlong]
,\[Closed\_Count\_1yProlong]
,\[Closed\_Count\_2yProlong]
,\[Closed\_Count\_3yProlong]
,\[Closed\_Count\_4yProlong]
,\[Closed\_Count\_5plusProlong]
,\[Closed\_Sum\_ProlongRub]
,\[Closed\_Sum\_NewNoProlong]
,\[Closed\_Sum\_1yProlong\_Rub]
,\[Closed\_Sum\_2yProlong\_Rub]
,\[Closed\_Sum\_3yProlong\_Rub]
,\[Closed\_Sum\_4yProlong\_Rub]
,\[Closed\_Sum\_5plusProlong\_Rub]
,\[Closed\_Sum\_ProlongRub\_int]
,\[Closed\_Sum\_NewNoProlong\_int]
,\[Closed\_Sum\_1yProlong\_Rub\_int]
,\[Closed\_Sum\_2yProlong\_Rub\_int]
,\[Closed\_Sum\_3yProlong\_Rub\_int]
,\[Closed\_Sum\_4yProlong\_Rub\_int]
,\[Closed\_Sum\_5plusProlong\_Rub\_int]
\-- ,\[Closed\_Dolya\_ProlongRub]
\--  ,\[Closed\_Dolya\_1yProlongRub]
\--  ,\[Closed\_Dolya\_2yProlongRub]
\--   ,\[Closed\_Dolya\_3yProlongRub]
\--  ,\[Closed\_Dolya\_4yProlongRub]
\--  ,\[Closed\_Dolya\_5plusProlongRub]
,\[WeightedRate\_Closed\_Overall]
,\[Closed\_WeightedRate\_All]
,\[Closed\_WeightedRate\_NewNoProlong]
,\[Closed\_WeightedRate\_1y]
,\[Closed\_WeightedRate\_2y]
,\[Closed\_WeightedRate\_3y]
,\[Closed\_WeightedRate\_4y]
,\[Closed\_WeightedRate\_5plus]
,\[Closed\_WeightedRate\_AllProlong]
\--    ,\[Dolya\_VyhodovRub]
FROM \[ALM\_TEST].\[WORK].\[prolongationAnalysisResult\_ISOPTION]

следовательно -в экселе у меня такие поля

вот код как я создал такую витрину

USE \[ALM\_TEST]
GO

/\*\*\*\*\*\* Object:  View \[WORK].\[vw\_prolongationAnalysis]    Script Date: 11.03.2025 8:38:39 \*\*\*\*\*\*/
SET ANSI\_NULLS ON
GO

SET QUOTED\_IDENTIFIER ON
GO

/\* DELETE FROM  \[WORK].\[prolongationAnalysisResult\_ISOPTION]
WHERE MonthEnd >='2025-01-31'; \*/
WITH
ISOPT AS(
SELECT
b.\[CON\_ID],
CASE
WHEN isnull(b.\[OptionRate\_TRF], 0) <0 THEN 1
ELSE 0 -- Если продукт не найден в таблице маппинга
END AS \[IS\_OPTION]
FROM \[LIQUIDITY].\[liq].\[InterestsRateForDeposit] b with (nolock)
),
--

## -- 0) Генерация месяцев (MonthEnd)

cteMonths AS (
SELECT CONVERT(date, '2025-03-31') AS MonthEnd
UNION ALL
SELECT EOMONTH(DATEADD(MONTH, 1, MonthEnd))
FROM cteMonths
WHERE EOMONTH(DATEADD(MONTH, 1, MonthEnd)) <= '2025-04-30'
),

---

## -- A1) Открытые депозиты (DealsInMonthRates)

DealsInMonthRates AS (
SELECT
M.MonthEnd,
CAST(dc.CON\_ID AS BIGINT) AS CON\_ID,
dc.CLI\_ID,
dc.SEG\_NAME,
ISNULL(dc.PROD\_NAME,'Без типа') as PROD\_NAME,
dc.CUR,
dc.DT\_OPEN,
dc.DT\_CLOSE,
dc.BALANCE\_RUB,
dc.RATE,
DATEDIFF(DAY, dc.DT\_OPEN, dc.DT\_CLOSE\_PLAN) AS MATUR,
conv.NEW\_CONVENTION\_NAME,
DATEDIFF(DAY, dc.DT\_OPEN, dc.DT\_CLOSE) AS DaysLived,
\-- Переводим ставку в ежемесячную конвенцию
LIQUIDITY.liq.fnc\_IntRate(
dc.RATE,
conv.NEW\_CONVENTION\_NAME,
'monthly',
DATEDIFF(DAY, dc.DT\_OPEN, dc.DT\_CLOSE\_PLAN),
1
) AS ConvertedRate
FROM cteMonths M
JOIN \[LIQUIDITY].\[liq].\[DepositContract\_all] dc WITH (NOLOCK)
ON dc.DT\_OPEN >= DATEADD(DAY, 1, EOMONTH(M.MonthEnd, -1))
AND dc.DT\_OPEN <  DATEADD(DAY, 1, M.MonthEnd)
AND dc.CLI\_SUBTYPE = 'INDIV'
AND dc.PROD\_NAME   != 'Эскроу'
AND dc.convention != 'ON\_DEMAND'
AND dc.DT\_CLOSE\_PLAN != '4444-01-01'
AND DATEDIFF(DAY, dc.DT\_OPEN, dc.DT\_CLOSE) > 10
JOIN LIQUIDITY.\[liq].man\_CONVENTION conv WITH (NOLOCK)
ON dc.CONVENTION = conv.CONVENTION\_NAME
where CAST(dc.CON\_ID AS BIGINT)  not in (select CAST(CON\_ID AS BIGINT) from \[LIQUIDITY].\[liq].\[man\_FloatContracts] with (nolock))
),

---

## -- A2) Присоединяем бакеты срочности (DealsInMonthBucket)

DealsInMonthBucket AS (
SELECT
dmr.MonthEnd,
dmr.CON\_ID,
dmr.CLI\_ID,
dmr.SEG\_NAME,
dmr.PROD\_NAME,
dmr.CUR,
dmr.DT\_OPEN,
dmr.DT\_CLOSE,
dmr.BALANCE\_RUB,
dmr.RATE,
dmr.ConvertedRate,
dmr.MATUR,
dmr.DaysLived,
tg.TERM\_GROUP AS TermBucket,
bal.BALANCE\_GROUP AS BalanceBucket,
CAST(isnull(opt.\[IS\_OPTION],0) as nvarchar) as IS\_OPTION
FROM DealsInMonthRates dmr
LEFT JOIN \[ALM\_TEST].\[WORK].\[man\_TermGroup] tg WITH (NOLOCK)
ON dmr.MATUR >= tg.TERM\_FROM
AND dmr.MATUR <= tg.TERM\_TO
LEFT JOIN \[ALM\_TEST].\[WORK].\[man\_BalanceGroup] bal
ON dmr.BALANCE\_RUB >= bal.BALANCE\_FROM
AND dmr.BALANCE\_RUB < bal.BALANCE\_TO
LEFT JOIN ISOPT opt WITH (NOLOCK)
ON dmr.CON\_ID =opt.CON\_ID
),

---

\-- A3) (рекурсивный CTE)
\--    Можно использовать для глубины 5 (5+).
---------------------------------------------

RecursiveChain AS (
SELECT
CAST(dmb.CON\_ID AS BIGINT)  AS StartConId,
CAST(dmb.CON\_ID AS BIGINT) AS CurrentConId,
0           AS Lvl
FROM DealsInMonthBucket dmb

```
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
```

),

---

## -- A4) Получаем ProlongCount (0..5)

ChainDepth AS (
SELECT
rc.StartConId,
CASE WHEN MAX(rc.Lvl) > 5 THEN 5 ELSE MAX(rc.Lvl) END AS ProlongCount
FROM RecursiveChain rc
GROUP BY rc.StartConId
),

---

## -- A5) Находим (Lvl=1) для вычисления PrevBalance, PrevConvertedRate

FirstProlongation AS (
SELECT
r1.StartConId,
r1.CurrentConId AS Old1\_ConId
FROM RecursiveChain r1
WHERE r1.Lvl = 1
),
--

## -- A6) Берём поля предыдущего вклада (Balance, RATE->PrevConvertedRate)

PreviousDepositInfo AS (
SELECT
fp.StartConId,
old1.CON\_ID      AS Old1\_ConId,
old1.BALANCE\_RUB AS PrevBalance,
LIQUIDITY.liq.fnc\_IntRate(
old1.RATE,
conv\_prev.NEW\_CONVENTION\_NAME,
'monthly',
DATEDIFF(DAY, old1.DT\_OPEN, old1.DT\_CLOSE\_PLAN),
1
) AS PrevConvertedRate
FROM FirstProlongation fp
JOIN \[LIQUIDITY].\[liq].\[DepositContract\_all] old1 WITH (NOLOCK)
ON old1.CON\_ID = fp.Old1\_ConId
JOIN LIQUIDITY.\[liq].man\_CONVENTION conv\_prev WITH (NOLOCK)
ON old1.CONVENTION = conv\_prev.CONVENTION\_NAME
),

---

## -- A7) Собираем данные (DealsWithProlong)

DealsWithProlong AS (
SELECT
dmb.MonthEnd,
dmb.CON\_ID,
dmb.CLI\_ID,
dmb.SEG\_NAME,
dmb.PROD\_NAME,
dmb.CUR,
dmb.DT\_OPEN,
dmb.DT\_CLOSE,
dmb.BALANCE\_RUB,
dmb.RATE,
dmb.MATUR,
dmb.ConvertedRate,
dmb.TermBucket,
dmb.BalanceBucket,
dmb.IS\_OPTION,
dmb.DaysLived,
ISNULL(cd.ProlongCount, 0) AS ProlongCount,
ISNULL(pdi.PrevBalance, 0) AS PrevBalance,
ISNULL(pdi.PrevConvertedRate, 0) AS PrevConvertedRate
FROM DealsInMonthBucket dmb
LEFT JOIN ChainDepth cd
ON cd.StartConId = dmb.CON\_ID
LEFT JOIN PreviousDepositInfo pdi
ON pdi.StartConId = dmb.CON\_ID
),
--

## -- A8) Аггрегация по открытым (OpenAggregated)

OpenAggregated\_0 AS (
SELECT
MonthEnd,
CASE
WHEN SEG\_NAME = 'Розничный бизнес' THEN N'Розница'
WHEN SEG\_NAME = 'ДЧБО' THEN N'ЧБО'
ELSE N'Без сегментации'
END AS SegmentGrouping,
PROD\_NAME,
CUR AS CurrencyGrouping,
TermBucket AS TermBucketGrouping,
BalanceBucket AS BalanceBucketGrouping,
IS\_OPTION,

```
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
```

## ),

## -- A8) Аггрегация по открытым (OpenAggregated)

OpenAggregated AS (
SELECT
ISNULL(MonthEnd, '9999-12-31') AS MonthEnd,
ISNULL(SegmentGrouping, N'Все сегменты') AS SegmentGrouping,
ISNULL(PROD\_NAME, N'Все продукты')		 AS PROD\_NAME,
ISNULL(CurrencyGrouping, N'Все валюты')  AS CurrencyGrouping,
ISNULL(TermBucketGrouping, N'Все бакеты')AS TermBucketGrouping,
ISNULL(BalanceBucketGrouping, N'Все бакеты')AS BalanceBucketGrouping,
ISNULL(IS\_OPTION,N'Все признаки опциональности') as IS\_OPTION,
OpenedDeals,
Summ\_BalanceRub,
Count\_Prolong,
Count\_NewNoProlong,
Count\_1yProlong,
Count\_2yProlong,
Count\_3yProlong,
Count\_4yProlong,
Count\_5plusProlong,
Sum\_ProlongRub,
Sum\_NewNoProlong,
Sum\_1yProlong\_Rub,
Sum\_2yProlong\_Rub,
Sum\_3yProlong\_Rub,
Sum\_4yProlong\_Rub,
Sum\_5plusProlong\_Rub,
Sum\_RateWeighted,
Sum\_RateWeighted\_NewNoProlong,
Sum\_RateWeighted\_1y,
Sum\_RateWeighted\_2y,
Sum\_RateWeighted\_3y,
Sum\_RateWeighted\_4y,
Sum\_RateWeighted\_5plus,
Sum\_RateWeighted\_Prolong,
Sum\_PreviousBalance,
Sum\_PreviousRateWeighted

```
FROM OpenAggregated_0
```

## ),

## -- A9) Средневзвешенные ставки (OpenWeightedRates)

OpenWeightedRates AS (
SELECT
MonthEnd,
SegmentGrouping,
PROD\_NAME,
CurrencyGrouping,
TermBucketGrouping,
BalanceBucketGrouping,
IS\_OPTION,
\-- Общая
CASE WHEN SUM(Summ\_BalanceRub) > 0
THEN SUM(Sum\_RateWeighted) / SUM(Summ\_BalanceRub)
ELSE 0 END AS WeightedRate\_All,

```
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
```

## ),

\-- B1) Закрытые депозиты (ClosedDealsInMonthRates)
\--     с расчетом ConvertedRate
--------------------------------

ClosedDealsInMonthRates AS (
SELECT
M.MonthEnd,
CAST(dc.CON\_ID AS BIGINT) AS CON\_ID,
dc.SEG\_NAME,
ISNULL(dc.PROD\_NAME,'Без типа') as PROD\_NAME,
dc.CUR,
dc.DT\_OPEN,
dc.DT\_CLOSE,
dc.BALANCE\_RUB,
dc.RATE,
DATEDIFF(DAY, dc.DT\_OPEN, dc.DT\_CLOSE\_PLAN) AS MATUR,
conv.NEW\_CONVENTION\_NAME,
DATEDIFF(DAY, dc.DT\_OPEN, dc.DT\_CLOSE) AS DaysLived,
\-- Аналогичная логика конвертации ставки
LIQUIDITY.liq.fnc\_IntRate(
dc.RATE,
conv.NEW\_CONVENTION\_NAME,
'monthly',
DATEDIFF(DAY, dc.DT\_OPEN, dc.DT\_CLOSE\_PLAN),
1
) AS ConvertedRate
FROM cteMonths M
JOIN \[LIQUIDITY].\[liq].\[DepositContract\_all] dc WITH (NOLOCK)
ON dc.DT\_CLOSE >= DATEADD(DAY, 1, EOMONTH(M.MonthEnd, -1))
AND dc.DT\_CLOSE <  DATEADD(DAY, 1, M.MonthEnd)
AND dc.CLI\_SUBTYPE = 'INDIV'
AND dc.PROD\_NAME   != 'Эскроу'
AND dc.CONVENTION  != 'ON\_DEMAND'
AND dc.DT\_CLOSE\_PLAN != '4444-01-01'
AND DATEDIFF(DAY, dc.DT\_OPEN, dc.DT\_CLOSE) > 10
JOIN LIQUIDITY.\[liq].man\_CONVENTION conv WITH (NOLOCK)
ON dc.CONVENTION = conv.CONVENTION\_NAME
where CAST(dc.CON\_ID AS BIGINT) not in (select CAST(CON\_ID AS BIGINT) from \[LIQUIDITY].\[liq].\[man\_FloatContracts] with (nolock))
),

---

## -- B2) Присоединяем бакеты срочности (ClosedDealsInMonthBucket)

ClosedDealsInMonthBucket AS (
SELECT
cdm.MonthEnd,
cdm.CON\_ID,
cdm.SEG\_NAME,
cdm.PROD\_NAME,
cdm.CUR,
cdm.DT\_OPEN,
cdm.DT\_CLOSE,
cdm.BALANCE\_RUB,
CASE
WHEN cdm.CUR ='RUR' and CAST(isnull(opt.\[IS\_OPTION],0) as nvarchar) ='0'
THEN
CASE
when cdm.MATUR=cdm.DaysLived then cdm.BALANCE\_RUB\*(POWER(1+cdm.ConvertedRate/12,cdm.DaysLived/31)-1)
ELSE  cdm.BALANCE\_RUB\*(POWER(1+0.01/1200,cdm.DaysLived/31)-1)
END
ELSE 0
END as Int\_Balance\_RUB,
cdm.RATE,
cdm.ConvertedRate,
cdm.MATUR,
cdm.DaysLived,
tg.TERM\_GROUP AS TermBucket,
bal.BALANCE\_GROUP AS BalanceBucket,
CAST(isnull(opt.\[IS\_OPTION],0) as nvarchar) as IS\_OPTION
FROM ClosedDealsInMonthRates cdm
LEFT JOIN \[ALM\_TEST].\[WORK].\[man\_TermGroup] tg WITH (NOLOCK)
ON cdm.MATUR >= tg.TERM\_FROM
AND cdm.MATUR <= tg.TERM\_TO
LEFT JOIN \[ALM\_TEST].\[WORK].\[man\_BalanceGroup] bal
ON cdm.BALANCE\_RUB >= bal.BALANCE\_FROM
AND cdm.BALANCE\_RUB < bal.BALANCE\_TO
LEFT JOIN ISOPT opt WITH (NOLOCK)
ON cdm.CON\_ID =opt.CON\_ID
),

---

\-- B3) (по желанию) Рекурсивная логика для
\--     вычисления ProlongCount (0..5) для закрытых
---------------------------------------------------

ClosedRecursiveChain AS (
SELECT
CAST(dmb.CON\_ID AS BIGINT) AS StartConId,
CAST(dmb.CON\_ID AS BIGINT) AS CurrentConId,
0 AS Lvl
FROM ClosedDealsInMonthBucket dmb

```
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
```

),

ClosedChainDepth AS (
SELECT
rc.StartConId,
CASE WHEN MAX(rc.Lvl) > 5 THEN 5 ELSE MAX(rc.Lvl) END AS ProlongCount
FROM ClosedRecursiveChain rc
GROUP BY rc.StartConId
),

---

\-- B4) Собираем данные (ClosedDealsWithProlong),
\--     но нам НЕ НУЖНО info по предыдущему вкладу
--------------------------------------------------

ClosedDealsWithProlong AS (
SELECT
dmb.MonthEnd,
dmb.CON\_ID,
dmb.SEG\_NAME,
dmb.PROD\_NAME,
dmb.CUR,
dmb.DT\_OPEN,
dmb.DT\_CLOSE,
dmb.BALANCE\_RUB,
dmb.Int\_Balance\_RUB,
dmb.RATE,
dmb.MATUR,
dmb.ConvertedRate,
dmb.TermBucket,
dmb.DaysLived,
dmb.BalanceBucket,
dmb.IS\_OPTION,
ISNULL(cd.ProlongCount, 0) AS ProlongCount
FROM ClosedDealsInMonthBucket dmb
LEFT JOIN ClosedChainDepth cd
ON cd.StartConId = dmb.CON\_ID
),

---

\-- B5) Агрегация по закрытым (ClosedAggregated)
\--     с учетом ProlongCount (0..5)
------------------------------------

ClosedAggregated\_0 AS (
SELECT
MonthEnd,
CASE
WHEN SEG\_NAME = 'Розничный бизнес' THEN N'Розница'
WHEN SEG\_NAME = 'ДЧБО' THEN N'ЧБО'
ELSE N'Без сегментации'
END AS SegmentGrouping,
PROD\_NAME,
CUR AS CurrencyGrouping,
TermBucket AS TermBucketGrouping,
BalanceBucket AS BalanceBucketGrouping,
IS\_OPTION,
COUNT(\*) AS ClosedDeals,
SUM(BALANCE\_RUB) AS Summ\_ClosedBalanceRub,
SUM(Int\_Balance\_RUB) As Summ\_ClosedBalanceRub\_int,

```
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
```

## ),

\-- B5) Агрегация по закрытым (ClosedAggregated)
\--     с учетом ProlongCount (0..5)
------------------------------------

ClosedAggregated AS (
SELECT
ISNULL(MonthEnd, '9999-12-31') AS MonthEnd,
ISNULL(SegmentGrouping, N'Все сегменты') AS SegmentGrouping,
ISNULL(PROD\_NAME, N'Все продукты')		 AS PROD\_NAME,
ISNULL(CurrencyGrouping, N'Все валюты')  AS CurrencyGrouping,
ISNULL(TermBucketGrouping, N'Все бакеты')AS TermBucketGrouping,
ISNULL(BalanceBucketGrouping, N'Все бакеты')AS BalanceBucketGrouping,
ISNULL(IS\_OPTION,N'Все признаки опциональности') as IS\_OPTION,
ClosedDeals,
Summ\_ClosedBalanceRub,
Summ\_ClosedBalanceRub\_int,
\-- Аналогично: считаем количество сделок, суммы в разрезе
Closed\_Count\_Prolong,
Closed\_Count\_NewNoProlong,
Closed\_Count\_1yProlong,
Closed\_Count\_2yProlong,
Closed\_Count\_3yProlong,
Closed\_Count\_4yProlong,
Closed\_Count\_5plusProlong,

```
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
```

),

---

\-- B6) Средневзвешенные ставки (ClosedWeightedRates)
\--     по уровню пролонгации (0..5)
------------------------------------

ClosedWeightedRates AS (
SELECT
MonthEnd,
SegmentGrouping,
PROD\_NAME,
CurrencyGrouping,
TermBucketGrouping,
BalanceBucketGrouping,
IS\_OPTION,

```
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
```

## ),

\-- Финальный Шаг: объединяем все CTE
\-- (OpenAggregated, OpenWeightedRates,
\--  ClosedAggregated, ClosedWeightedRates)
\-- через FULL OUTER JOIN, а затем
\-- делаем GROUP BY, чтобы "слить" дубли
----------------------------------------

FullJoin AS
(
SELECT
\--------------------------------
\-- Ключи "Raw\*" для дальнейшей группировки
\--------------------------------
COALESCE(o.MonthEnd, ow\.MonthEnd, c.MonthEnd, cw\.MonthEnd) AS RawMonthEnd,
COALESCE(o.SegmentGrouping, ow\.SegmentGrouping, c.SegmentGrouping, cw\.SegmentGrouping) AS RawSegmentGrouping,
COALESCE(o.PROD\_NAME,ow\.PROD\_NAME,c.PROD\_NAME,cw\.PROD\_NAME) AS RawPROD\_NAME,
COALESCE(o.CurrencyGrouping, ow\.CurrencyGrouping, c.CurrencyGrouping, cw\.CurrencyGrouping) AS RawCurrencyGrouping,
COALESCE(o.TermBucketGrouping, ow\.TermBucketGrouping, c.TermBucketGrouping, cw\.TermBucketGrouping) AS RawTermBucketGrouping,
COALESCE(o.BalanceBucketGrouping,ow\.BalanceBucketGrouping,c.BalanceBucketGrouping,cw\.BalanceBucketGrouping) AS RawBalanceBucketGrouping,
COALESCE(o.IS\_OPTION, ow\.IS\_OPTION,c.IS\_OPTION,cw\.IS\_OPTION) AS RawIS\_OPTION,
\--------------------------------
\-- Поля по ОТКРЫТЫМ ВКЛАДАМ
\--------------------------------
o.OpenedDeals,
o.Summ\_BalanceRub,

```
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
```

)

---

INSERT INTO \[WORK].\[prolongationAnalysisResult\_ISOPTION]
SELECT
\-- Заменяем NULL на "Все ...":
RawMonthEnd AS MonthEnd,
RawSegmentGrouping AS SegmentGrouping,
RawPROD\_NAME	 AS PROD\_NAME,
FullJoin.RawCurrencyGrouping  AS CurrencyGrouping,
FullJoin.RawTermBucketGrouping AS TermBucketGrouping,
FullJoin.RawBalanceBucketGrouping as BalanceBucketGrouping,
FullJoin.RawIS\_OPTION as IS\_OPTION,
OpenedDeals                  AS OpenedDeals,
Summ\_BalanceRub              AS Opened\_Summ\_BalanceRub,
Count\_Prolong               AS Opened\_Count\_Prolong,
Count\_NewNoProlong           AS Opened\_Count\_NewNoProlong,
Count\_1yProlong              AS Opened\_Count\_1yProlong,
Count\_2yProlong              AS Opened\_Count\_2yProlong,
Count\_3yProlong              AS Opened\_Count\_3yProlong,
Count\_4yProlong              AS Opened\_Count\_4yProlong,
Count\_5plusProlong           AS Opened\_Count\_5plusProlong,

```
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
```

FROM FullJoin
/\*GROUP BY
RawMonthEnd,
RawSegmentGrouping,
RawPROD\_NAME,
RawCurrencyGrouping,
RawTermBucketGrouping
\*/
ORDER BY
RawMonthEnd;

GO

2	Общая пролонгация	=ЕСЛИОШИБКА(Opened\_Sum\_ProlongRub/(Summ\_ClosedBalanceRub+Summ\_ClosedBalanceRub\_int);0)
4	1-ая автопролонгация	=ЕСЛИОШИБКА(Opened\_Sum\_1yProlong\_Rub/(Closed\_Sum\_NewNoProlong+Closed\_Sum\_NewNoProlong\_int);0)

6	2-ая автопролонгация 	=ЕСЛИОШИБКА(Opened\_Sum\_2yProlong\_Rub/(Closed\_Sum\_1yProlong\_Rub+Closed\_Sum\_1yProlong\_Rub\_int);0)

8	3-ая автопролонгация 	=ЕСЛИОШИБКА(Opened\_Sum\_3yProlong\_Rub/(Closed\_Sum\_2yProlong\_Rub+Closed\_Sum\_2yProlong\_Rub\_int);0)

твоя задача детально и хорошо запомнить структуру данных таблиц
