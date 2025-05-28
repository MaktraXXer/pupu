/*=====================================================================
  LineDump (апрель-2025): полные построчные данные открытых/закрытых ФЛ
  формулы и маппинги 1-в-1 с витриной prolongationAnalysisResult_ISOPTION
=====================================================================*/
USE ALM_TEST;
GO
SET NOCOUNT ON;
SET ANSI_NULLS , QUOTED_IDENTIFIER ON;
GO

/* ---------- параметры периода ---------- */
DECLARE @MonthEnd  date = '2025-04-30';
DECLARE @MonthBeg  date = DATEADD(DAY,1,EOMONTH(@MonthEnd,-1));
DECLARE @DayWindow int  = 3;       /* ±N дней для поиска базы */

/* ====================================================================
   Часть A :  О Т К Р Ы Т Ы Е   Д Е П О З И Т Ы
   ================================================================== */
WITH
/* ---------- 0. справочник «признак опциональности» ------------------*/
ISOPT AS (
    SELECT CAST(CON_ID AS bigint)            AS CON_ID,
           CASE WHEN ISNULL(OptionRate_TRF,0) < 0 THEN 1 ELSE 0 END AS IS_OPTION
    FROM   LIQUIDITY.liq.InterestsRateForDeposit WITH (NOLOCK)
),
/* ---------- 1. целевой месяц --------------------------------------- */
cteMonths AS ( SELECT @MonthEnd AS MonthEnd ),
/* ---------- 2. исходные открытые договоры + конвертация ставки ------ */
DealsInMonthRates AS (
    SELECT
        M.MonthEnd,
        dc.CON_ID,
        dc.CLI_ID,
        dc.SEG_NAME,
        ISNULL(dc.PROD_NAME,'Без типа')                  AS PROD_NAME,
        dc.CUR,
        dc.DT_OPEN,
        dc.DT_CLOSE,
        dc.BALANCE_RUB,
        dc.RATE,
        DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE_PLAN)        AS MATUR,
        conv.NEW_CONVENTION_NAME,
        DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE)             AS DaysLived,
        /* ставка, приведённая к monthly (со SNAP и коэффициентами) */
        ISNULL(snap.MonthlyCONV_RATE,
               LIQUIDITY.liq.fnc_IntRate(
                   (dc.RATE+0.0048)/0.9525,
                   conv.NEW_CONVENTION_NAME,
                   'monthly',
                   DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE_PLAN),
                   1
               )*0.9525-0.0048)                          AS ConvertedRate
    FROM   cteMonths               M
    JOIN   LIQUIDITY.liq.DepositContract_all dc  WITH (NOLOCK)
           ON dc.DT_OPEN >= @MonthBeg
          AND dc.DT_OPEN <  DATEADD(DAY,1,@MonthEnd)
          AND dc.CLI_SUBTYPE = 'INDIV'
          AND dc.PROD_NAME  <> 'Эскроу'
          AND dc.CONVENTION <> 'ON_DEMAND'
          AND dc.DT_CLOSE_PLAN <> '4444-01-01'
          AND DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE) > 10
    JOIN   LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
           ON conv.CONVENTION_NAME = dc.CONVENTION
    LEFT   JOIN ALM_TEST.WORK.DepositInterestsRateSnap snap WITH (NOLOCK)
           ON snap.CON_ID = dc.CON_ID
    WHERE  dc.CON_ID NOT IN (SELECT CON_ID
                             FROM LIQUIDITY.liq.man_FloatContracts WITH (NOLOCK))
),
/* ---------- 3. бакеты срочности/сумм-------------------------------- */
DealsInMonthBucket AS (
    SELECT dmr.*,
           tg.TERM_GROUP             AS TermBucket,
           bal.BALANCE_GROUP         AS BalanceBucket,
           CAST(ISNULL(opt.IS_OPTION,0) AS nvarchar) AS IS_OPTION
    FROM  DealsInMonthRates dmr
    LEFT JOIN ALM_TEST.WORK.man_TermGroup tg  WITH (NOLOCK)
           ON dmr.MATUR BETWEEN tg.TERM_FROM AND tg.TERM_TO
    LEFT JOIN ALM_TEST.WORK.man_BalanceGroup bal WITH (NOLOCK)
           ON dmr.BALANCE_RUB >= bal.BALANCE_FROM
          AND dmr.BALANCE_RUB <  bal.BALANCE_TO
    LEFT JOIN ISOPT opt WITH (NOLOCK)
           ON opt.CON_ID = dmr.CON_ID
),
/* ---------- 4. рекурсивно ищем цепочку пролонгаций ------------------ */
RecursiveChain AS (
    SELECT CAST(CON_ID AS bigint) AS StartConId,
           CAST(CON_ID AS bigint) AS CurrentConId,
           0 AS Lvl
    FROM DealsInMonthBucket
    UNION ALL
    SELECT rc.StartConId,
           CAST(p.CON_ID_REL AS bigint),
           rc.Lvl + 1
    FROM   RecursiveChain rc
    JOIN   ALM.ehd.conrel_prolongations p WITH (NOLOCK)
           ON p.CON_ID = rc.CurrentConId
          AND p.CON_REL_TYPE='PREVIOUS'
    WHERE  rc.Lvl < 5
),
ChainDepth AS (
    SELECT StartConId,
           CASE WHEN MAX(Lvl)>5 THEN 5 ELSE MAX(Lvl) END AS ProlongCount
    FROM   RecursiveChain
    GROUP BY StartConId
),
/* ---------- 5. первый «старый» вклад (Lvl=1) ------------------------ */
FirstProlong AS (
    SELECT StartConId, MIN(CurrentConId) AS Old1
    FROM   RecursiveChain
    WHERE  Lvl = 1
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
               1)                                AS PrevConvertedRate
    FROM FirstProlong fp
    JOIN LIQUIDITY.liq.DepositContract_all old WITH (NOLOCK)
           ON old.CON_ID = fp.Old1
    JOIN LIQUIDITY.liq.man_CONVENTION conv_prev WITH (NOLOCK)
           ON conv_prev.CONVENTION_NAME = old.CONVENTION
),
/* ---------- 6. объединяем всё выше ------------------------------- */
DealsWithProlong AS (
    SELECT dmb.*,
           ISNULL(cd.ProlongCount,0)              AS ProlongCount,
           ISNULL(pi.PrevBalance,0)               AS PrevBalance,
           ISNULL(pi.PrevConvertedRate,0)         AS PrevConvertedRate
    FROM  DealsInMonthBucket dmb
    LEFT JOIN ChainDepth cd ON cd.StartConId  = dmb.CON_ID
    LEFT JOIN PrevInfo  pi ON pi.StartConId   = dmb.CON_ID
),
/* ---------- 7. логика базовой ставки / дисконта ------------------- */
/*  (точно так же, как в исходной витрине)                           */
BalanceBuckets AS (
    SELECT BALANCE_GROUP, BALANCE_FROM, BALANCE_TO,
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
           AVG(dwp.ConvertedRate) AS BaseRate
    FROM DealsWithProlong dwp
    JOIN BalanceBuckets bb ON bb.BALANCE_GROUP = dwp.BalanceBucket
    WHERE dwp.ProlongCount = 0
    GROUP BY dwp.DT_OPEN,dwp.SEG_NAME,dwp.PROD_NAME,dwp.CUR,
             dwp.TermBucket,bb.BucketOrder,dwp.BalanceBucket,
             dwp.IS_OPTION,dwp.NEW_CONVENTION_NAME
),
DealKeys AS (
    SELECT DISTINCT
           d.DT_OPEN,d.SEG_NAME,d.PROD_NAME,d.CUR,
           d.TermBucket,d.IS_OPTION,
           bb.BucketOrder,bb.BALANCE_TO              AS BucketUpper,
           d.BalanceBucket,d.NEW_CONVENTION_NAME
    FROM DealsWithProlong d
    JOIN BalanceBuckets bb ON bb.BALANCE_GROUP=d.BalanceBucket
),
BestBase AS (
    SELECT dk.*,
           bc.BaseRate,
           ROW_NUMBER() OVER(
               PARTITION BY dk.DT_OPEN,dk.SEG_NAME,dk.PROD_NAME,
                            dk.CUR,dk.TermBucket,dk.IS_OPTION,
                            dk.NEW_CONVENTION_NAME,dk.BalanceBucket
               ORDER BY
                   CASE WHEN bc.BaseRate IS NULL THEN 1 ELSE 0 END,
                   ABS(DATEDIFF(DAY,bc.DT_OPEN,dk.DT_OPEN)),
                   CASE WHEN dk.BucketUpper<=1500000
                        THEN dk.BucketOrder-ISNULL(bbSrc.BucketOrder,dk.BucketOrder)
                        ELSE ISNULL(bbSrc.BucketOrder,dk.BucketOrder)-dk.BucketOrder
                   END,
                   CASE WHEN bc.DT_OPEN<dk.DT_OPEN THEN 0 ELSE 1 END
           ) rn
    FROM DealKeys dk
    LEFT JOIN BaseCandidates bc
           ON bc.SEG_NAME = dk.SEG_NAME
          AND bc.PROD_NAME= dk.PROD_NAME
          AND bc.CUR      = dk.CUR
          AND bc.TermBucket = dk.TermBucket
          AND bc.IS_OPTION = dk.IS_OPTION
          AND bc.NEW_CONVENTION_NAME = dk.NEW_CONVENTION_NAME
          AND ABS(DATEDIFF(DAY,bc.DT_OPEN,dk.DT_OPEN))<=@DayWindow
          AND ((dk.BucketUpper<=1500000 AND bc.BucketOrder<=dk.BucketOrder)
            OR (dk.BucketUpper>1500000  AND bc.BucketOrder>=dk.BucketOrder))
    LEFT JOIN BalanceBuckets bbSrc ON bbSrc.BALANCE_GROUP = bc.BalanceBucket
),
DailyBaseRate AS ( SELECT * FROM BestBase WHERE rn=1 ),
DealsWithDiscount AS (
    SELECT dwp.*,
           CASE WHEN dwp.ProlongCount=0
                THEN dwp.ConvertedRate
                ELSE dbr.BaseRate END                 AS BaseRate,
           CASE WHEN dwp.ProlongCount=0
                THEN 0
                WHEN dbr.BaseRate IS NULL
                THEN NULL
                ELSE dwp.ConvertedRate - dbr.BaseRate END AS Discount
    FROM DealsWithProlong dwp
    LEFT JOIN DailyBaseRate dbr
           ON  dbr.DT_OPEN             = dwp.DT_OPEN
          AND dbr.SEG_NAME            = dwp.SEG_NAME
          AND dbr.PROD_NAME           = dwp.PROD_NAME
          AND dbr.CUR                 = dwp.CUR
          AND dbr.TermBucket          = dwp.TermBucket
          AND dbr.BalanceBucket       = dwp.BalanceBucket
          AND dbr.IS_OPTION           = dwp.IS_OPTION
          AND dbr.NEW_CONVENTION_NAME = dwp.NEW_CONVENTION_NAME
),
/* ---------- 8. построчный слой для открытых ------------------------ */
OpenLine AS (
    SELECT
        d.MonthEnd,
        CASE WHEN d.SEG_NAME='Розничный бизнес' THEN N'Розница'
             WHEN d.SEG_NAME='ДЧБО'             THEN N'ЧБО'
             ELSE N'Без сегментации' END        AS SegmentGrouping,
        d.PROD_NAME,
        d.CUR                                   AS CurrencyGrouping,
        d.TermBucket                            AS TermBucketGrouping,
        d.BalanceBucket                         AS BalanceBucketGrouping,
        CAST(d.IS_OPTION AS nvarchar)           AS IS_OPTION,

        /* служебные */
        'Opened'        AS DealType,
        d.CON_ID,
        d.CLI_ID,
        d.DT_OPEN,
        d.DT_CLOSE_PLAN AS DT_Matur,
        d.BALANCE_RUB   AS BalanceRub,
        d.ConvertedRate,
        d.BaseRate,
        d.Discount,
        d.ProlongCount,

        /* Count_-flags */
        CASE WHEN d.ProlongCount>0 THEN 1 ELSE 0 END AS Count_Prolong,
        CASE WHEN d.ProlongCount=0 THEN 1 ELSE 0 END AS Count_NewNoProlong,
        CASE WHEN d.ProlongCount=1 THEN 1 ELSE 0 END AS Count_1yProlong,
        CASE WHEN d.ProlongCount=2 THEN 1 ELSE 0 END AS Count_2yProlong,
        CASE WHEN d.ProlongCount=3 THEN 1 ELSE 0 END AS Count_3yProlong,
        CASE WHEN d.ProlongCount=4 THEN 1 ELSE 0 END AS Count_4yProlong,
        CASE WHEN d.ProlongCount>=5 THEN 1 ELSE 0 END AS Count_5plusProlong,

        /* Sum_… */
        CASE WHEN d.ProlongCount>0 THEN d.BALANCE_RUB ELSE 0 END AS Sum_ProlongRub,
        CASE WHEN d.ProlongCount=0 THEN d.BALANCE_RUB ELSE 0 END AS Sum_NewNoProlong,
        CASE WHEN d.ProlongCount=1 THEN d.BALANCE_RUB ELSE 0 END AS Sum_1yProlong_Rub,
        CASE WHEN d.ProlongCount=2 THEN d.BALANCE_RUB ELSE 0 END AS Sum_2yProlong_Rub,
        CASE WHEN d.ProlongCount=3 THEN d.BALANCE_RUB ELSE 0 END AS Sum_3yProlong_Rub,
        CASE WHEN d.ProlongCount=4 THEN d.BALANCE_RUB ELSE 0 END AS Sum_4yProlong_Rub,
        CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_RUB ELSE 0 END AS Sum_5plusProlong_Rub,

        d.BALANCE_RUB*d.ConvertedRate           AS Sum_RateWeighted,
        CASE WHEN d.ProlongCount=0  THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_NewNoProlong,
        CASE WHEN d.ProlongCount=1  THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_1y,
        CASE WHEN d.ProlongCount=2  THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_2y,
        CASE WHEN d.ProlongCount=3  THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_3y,
        CASE WHEN d.ProlongCount=4  THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_4y,
        CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_5plus,

        d.BALANCE_RUB*d.Discount               AS Sum_Discount,
        CASE WHEN d.ProlongCount=0  THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_NewNoProlong,
        CASE WHEN d.ProlongCount=1  THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_1y,
        CASE WHEN d.ProlongCount=2  THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_2y,
        CASE WHEN d.ProlongCount=3  THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_3y,
        CASE WHEN d.ProlongCount=4  THEN d.BALANCE_Rуб*d.Discount ELSE 0 END AS Sum_Discount_4y,
        CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_5plus
    FROM DealsWithDiscount d
),
/* ====================================================================
   Часть B :  З А К Р Ы Т Ы Е
   ================================================================== */
ClosedDealsInMonthRates AS (
    SELECT
        M.MonthEnd,
        CAST(dc.CON_ID AS bigint)                AS CON_ID,
        dc.SEG_NAME,
        ISNULL(dc.PROD_NAME,'Без типа')          AS PROD_NAME,
        dc.CUR,
        dc.DT_OPEN,
        dc.DT_CLOSE,
        dc.BALANCE_RUB,
        dc.RATE,
        DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE_PLAN) AS MATUR,
        conv.NEW_CONVENTION_NAME,
        DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE)      AS DaysLived,
        ISNULL(snap.MonthlyCONV_RATE,
               LIQUIDITY.liq.fnc_IntRate(
                   (dc.RATE+0.0048)/0.9525,
                   conv.NEW_CONVENTION_NAME,
                   'monthly',
                   DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE_PLAN),
                   1)*0.9525-0.0048)              AS ConvertedRate
    FROM  cteMonths M
    JOIN  LIQUIDITY.liq.DepositContract_all dc WITH (NOLOCK)
          ON dc.DT_CLOSE >= @MonthBeg
         AND dc.DT_CLOSE <  DATEADD(DAY,1,@MonthEnd)
         AND dc.CLI_SUBTYPE='INDIV'
         AND dc.PROD_NAME <> 'Эскроу'
         AND dc.CONVENTION<> 'ON_DEMAND'
         AND dc.DT_CLOSE_PLAN <> '4444-01-01'
         AND DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE) > 10
    JOIN  LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
          ON conv.CONVENTION_NAME = dc.CONVENTION
    LEFT  JOIN ALM_TEST.WORK.DepositInterestsRateSnap snap WITH (NOLOCK)
          ON snap.CON_ID = dc.CON_ID
    WHERE CAST(dc.CON_ID AS bigint) NOT IN
          (SELECT CON_ID FROM LIQUIDITY.liq.man_FloatContracts WITH (NOLOCK))
),
ClosedDealsInMonthBucket AS (
    SELECT
        cdm.*,
        tg.TERM_GROUP          AS TermBucket,
        bal.BALANCE_GROUP      AS BalanceBucket,
        CAST(ISNULL(opt.IS_OPTION,0) AS nvarchar) AS IS_OPTION,
        CASE WHEN cdm.CUR='RUR' AND ISNULL(opt.IS_OPTION,0)=0
             THEN CASE WHEN cdm.MATUR=cdm.DaysLived
                       THEN cdm.BALANCE_RUB*(POWER(1+cdm.ConvertedRate/12,
                                                  cdm.DaysLived/31)-1)
                       ELSE cdm.BALANCE_RUB*(POWER(1+0.01/1200,
                                                  cdm.DaysLived/31)-1)
                  END
             ELSE 0 END            AS Int_Balance_RUB
    FROM ClosedDealsInMonthRates cdm
    LEFT JOIN ALM_TEST.WORK.man_TermGroup tg WITH (NOLOCK)
          ON cdm.MATUR BETWEEN tg.TERM_FROM AND tg.TERM_TO
    LEFT JOIN ALM_TEST.WORK.man_BalanceGroup bal WITH (NOLOCK)
          ON cdm.BALANCE_RUB >= bal.BALANCE_FROM
         AND cdm.BALANCE_RUB <  bal.BALANCE_TO
    LEFT JOIN ISOPT opt WITH (NOLOCK)
          ON opt.CON_ID = cdm.CON_ID
),
ClosedRecursiveChain AS (
    SELECT CAST(CON_ID AS bigint) AS StartConId,
           CAST(CON_ID AS bigint) AS CurrentConId,
           0 AS Lvl
    FROM ClosedDealsInMonthBucket
    UNION ALL
    SELECT rc.StartConId,
           CAST(p.CON_ID_REL AS bigint),
           rc.Lvl+1
    FROM   ClosedRecursiveChain rc
    JOIN   ALM.ehd.conrel_prolongations p WITH (NOLOCK)
           ON p.CON_ID = rc.CurrentConId
          AND p.CON_REL_TYPE='PREVIOUS'
    WHERE  rc.Lvl < 5
),
ClosedChainDepth AS (
    SELECT StartConId,
           CASE WHEN MAX(Lvl)>5 THEN 5 ELSE MAX(Lvl) END AS ProlongCount
    FROM ClosedRecursiveChain
    GROUP BY StartConId
),
ClosedDealsWithProlong AS (
    SELECT dmb.*,
           ISNULL(ccd.ProlongCount,0) AS ProlongCount
    FROM ClosedDealsInMonthBucket dmb
    LEFT JOIN ClosedChainDepth ccd ON ccd.StartConId = dmb.CON_ID
),
ClosedLine AS (
    SELECT
        d.MonthEnd,
        CASE WHEN d.SEG_NAME='Розничный бизнес' THEN N'Розница'
             WHEN d.SEG_NAME='ДЧБО'             THEN N'ЧБО'
             ELSE N'Без сегментации' END        AS SegmentGrouping,
        d.PROD_NAME,
        d.CUR                                   AS CurrencyGrouping,
        d.TermBucket                            AS TermBucketGrouping,
        d.BalanceBucket                         AS BalanceBucketGrouping,
        CAST(d.IS_OPTION AS nvarchar)           AS IS_OPTION,

        'Closed'         AS DealType,
        d.CON_ID,
        NULL             AS CLI_ID,
        d.DT_OPEN,
        d.DT_CLOSE       AS DT_Matur,
        d.BALANCE_RUB    AS BalanceRub,
        d.ConvertedRate,
        NULL             AS BaseRate,
        NULL             AS Discount,
        d.ProlongCount,

        /* Count */
        CASE WHEN d.ProlongCount>0 THEN 1 ELSE 0 END AS Closed_Count_Prolong,
        CASE WHEN d.ProlongCount=0 THEN 1 ELSE 0 END AS Closed_Count_NewNoProlong,
        CASE WHEN d.ProlongCount=1 THEN 1 ELSE 0 END AS Closed_Count_1yProlong,
        CASE WHEN d.ProlongCount=2 THEN 1 ELSE 0 END AS Closed_Count_2yProlong,
        CASE WHEN d.ProlongCount=3 THEN 1 ELSE 0 END AS Closed_Count_3yProlong,
        CASE WHEN d.ProlongCount=4 THEN 1 ELSE 0 END AS Closed_Count_4yProlong,
        CASE WHEN d.ProlongCount>=5 THEN 1 ELSE 0 END AS Closed_Count_5plusProlong,

        /* Sum */
        CASE WHEN d.ProlongCount>0 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_ProlongRub,
        CASE WHEN d.ProlongCount=0 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_NewNoProlong,
        CASE WHEN d.ProlongCount=1 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_1yProlong_Rub,
        CASE WHEN d.ProlongCount=2 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_2yProlong_Rub,
        CASE WHEN d.ProlongCount=3 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_3yProlong_Rub,
        CASE WHEN d.ProlongCount=4 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_4yProlong_Rub,
        CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_5plusProlong_Rub,

        d.Int_Balance_RUB                                    AS Int_Balance_RUB,
        CASE WHEN d.ProlongCount>0 THEN d.Int_Balance_RUB ELSE 0 END AS Closed_Sum_ProlongRub_int,
        CASE WHEN d.ProlongCount=0 THEN d.Int_Balance_RUB ELSE 0 END AS Closed_Sum_NewNoProlong_int
    FROM ClosedDealsWithProlong d
)

/* ====================================================================
   И Т О Г  (opened ∪ closed)
   ================================================================== */
SELECT * 
INTO   #LineDump202504       --<<- временная таблица; уберите INTO если не нужно
FROM   OpenLine
UNION ALL
SELECT * 
FROM   ClosedLine;

-- посмотреть результат
SELECT TOP (100) * FROM #LineDump202504 ORDER BY DealType, PROD_NAME, CON_ID;
