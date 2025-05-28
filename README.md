/*=====================================================================
  LineDump (апрель-2025) – построчные данные открытых и закрытых вкладов
  Все формулы/разбивки = витрина WORK.prolongationAnalysisResult_ISOPTION
=====================================================================*/
USE ALM_TEST;
GO
SET NOCOUNT ON;
SET ANSI_NULLS , QUOTED_IDENTIFIER ON;
GO

/* ---------- период отчёта ---------- */
DECLARE @MonthEnd  date = '2025-04-30';
DECLARE @MonthBeg  date = DATEADD(DAY,1,EOMONTH(@MonthEnd,-1));
DECLARE @DayWindow int  = 3;      /* ±N дней поиска базовой ставки */

/* ======================================================================= */
/*  A.  О Т К Р Ы Т Ы Е                                                    */
/* ======================================================================= */
WITH
/* --- 0. признак «опционный» ------------------------------------------- */
ISOPT AS (
    SELECT CAST(CON_ID AS bigint) AS CON_ID,
           CASE WHEN ISNULL(OptionRate_TRF,0) < 0 THEN 1 ELSE 0 END AS IS_OPTION
    FROM   LIQUIDITY.liq.InterestsRateForDeposit WITH (NOLOCK)
),
/* --- 1. целевой месяц ------------------------------------------------- */
cteMonths AS ( SELECT @MonthEnd AS MonthEnd ),
/* --- 2. исходные сделки + ConvertedRate + DT_CLOSE_PLAN --------------- */
DealsInMonthRates AS (
    SELECT
        M.MonthEnd,
        dc.CON_ID,
        dc.CLI_ID,
        dc.SEG_NAME,
        ISNULL(dc.PROD_NAME,'Без типа')              AS PROD_NAME,
        dc.CUR,
        dc.DT_OPEN,
        dc.DT_CLOSE,
        dc.DT_CLOSE_PLAN,                            -- ← нужно далее
        dc.BALANCE_RUB,
        dc.RATE,
        DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE_PLAN)    AS MATUR,
        conv.NEW_CONVENTION_NAME,
        DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE)         AS DaysLived,
        ISNULL(snap.MonthlyCONV_RATE,
               LIQUIDITY.liq.fnc_IntRate(
                   (dc.RATE+0.0048)/0.9525,
                   conv.NEW_CONVENTION_NAME,
                   'monthly',
                   DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE_PLAN),
                   1
               )*0.9525 - 0.0048)                   AS ConvertedRate
    FROM   cteMonths M
    JOIN   LIQUIDITY.liq.DepositContract_all dc WITH (NOLOCK)
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
/* --- 3. бакеты срочности / суммы ------------------------------------- */
DealsInMonthBucket AS (
    SELECT dmr.*,
           tg.TERM_GROUP         AS TermBucket,
           bal.BALANCE_GROUP     AS BalanceBucket,
           CAST(ISNULL(opt.IS_OPTION,0) AS nvarchar) AS IS_OPTION
    FROM  DealsInMonthRates dmr
    LEFT JOIN ALM_TEST.WORK.man_TermGroup      tg  WITH (NOLOCK)
           ON dmr.MATUR  BETWEEN tg.TERM_FROM AND tg.TERM_TO
    LEFT JOIN ALM_TEST.WORK.man_BalanceGroup   bal WITH (NOLOCK)
           ON dmr.BALANCE_RUB >= bal.BALANCE_FROM
          AND dmr.BALANCE_Rуб <  bal.BALANCE_TO
    LEFT JOIN ISOPT opt WITH (NOLOCK)
           ON opt.CON_ID = dmr.CON_ID
),
/* --- 4. рекурсивная цепочка пролонгаций ------------------------------ */
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
/* --- 5. предыдущий вклад (Lvl=1) ------------------------------------ */
FirstProlong AS (
    SELECT StartConId, MIN(CurrentConId) AS Old1
    FROM   RecursiveChain
    WHERE  Lvl = 1
    GROUP BY StartConId
),
PrevInfo AS (
    SELECT fp.StartConId,
           old.BALANCE_RUB                                                  AS PrevBalance,
           LIQUIDITY.liq.fnc_IntRate(old.RATE,
                                     conv.NEW_CONVENTION_NAME,
                                     'monthly',
                                     DATEDIFF(DAY,old.DT_OPEN,old.DT_CLOSE_PLAN),
                                     1)                                     AS PrevConvertedRate
    FROM FirstProlong fp
    JOIN LIQUIDITY.liq.DepositContract_all old WITH (NOLOCK)
           ON old.CON_ID = fp.Old1
    JOIN LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
           ON conv.CONVENTION_NAME = old.CONVENTION
),
/* --- 6. объединяем всё ---------------------------------------------- */
DealsWithProlong AS (
    SELECT  dmb.*,
            ISNULL(cd.ProlongCount,0)      AS ProlongCount,
            ISNULL(pi.PrevBalance,0)       AS PrevBalance,
            ISNULL(pi.PrevConvertedRate,0) AS PrevConvertedRate
    FROM DealsInMonthBucket dmb
    LEFT JOIN ChainDepth cd  ON cd.StartConId = dmb.CON_ID
    LEFT JOIN PrevInfo   pi  ON pi.StartConId = dmb.CON_ID
),
/* --- 7. вычисляем BaseRate / Discount (правила витрины) -------------- */
--------------------------------------------------------------------------------
-- вспомогательные CTE (бакеты сумм, поисковые ключи, BestBase и др.)
--------------------------------------------------------------------------------
BalanceBuckets AS (
    SELECT BALANCE_GROUP, BALANCE_FROM, BALANCE_TO,
           ROW_NUMBER() OVER(ORDER BY BALANCE_FROM) AS BucketOrder
    FROM ALM_TEST.WORK.man_BalanceGroup
),
BaseCandidates AS (
    SELECT d.DT_OPEN,
           d.SEG_NAME, d.PROD_NAME, d.CUR,
           d.TermBucket,
           bb.BucketOrder,
           d.BalanceBucket,
           d.IS_OPTION,
           d.NEW_CONVENTION_NAME,
           AVG(d.ConvertedRate) AS BaseRate
    FROM DealsWithProlong d
    JOIN BalanceBuckets bb ON bb.BALANCE_GROUP = d.BalanceBucket
    WHERE d.ProlongCount = 0
    GROUP BY d.DT_OPEN,d.SEG_NAME,d.PROD_NAME,d.CUR,
             d.TermBucket,bb.BucketOrder,d.BalanceBucket,
             d.IS_OPTION,d.NEW_CONVENTION_NAME
),
DealKeys AS (
    SELECT DISTINCT
           d.DT_OPEN,d.SEG_NAME,d.PROD_NAME,d.CUR,
           d.TermBucket,d.IS_OPTION,
           bb.BucketOrder, bb.BALANCE_TO AS BucketUpper,
           d.BalanceBucket,d.NEW_CONVENTION_NAME
    FROM DealsWithProlong d
    JOIN BalanceBuckets bb ON bb.BALANCE_GROUP = d.BalanceBucket
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
           ) AS rn
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
    LEFT JOIN BalanceBuckets bbSrc
           ON bbSrc.BALANCE_GROUP = bc.BalanceBucket
),
DailyBaseRate AS ( SELECT * FROM BestBase WHERE rn=1 ),
DealsWithDiscount AS (
    SELECT d.*,
           CASE WHEN d.ProlongCount=0 THEN d.ConvertedRate
                ELSE dbr.BaseRate END                                 AS BaseRate,
           CASE WHEN d.ProlongCount=0 THEN 0
                WHEN dbr.BaseRate IS NULL THEN NULL
                ELSE d.ConvertedRate - dbr.BaseRate END              AS Discount
    FROM DealsWithProlong d
    LEFT JOIN DailyBaseRate dbr
           ON  dbr.DT_OPEN            = d.DT_OPEN
          AND dbr.SEG_NAME           = d.SEG_NAME
          AND dbr.PROD_NAME          = d.PROD_NAME
          AND dbr.CUR                = d.CUR
          AND dbr.TermBucket         = d.TermBucket
          AND dbr.BalanceBucket      = d.BalanceBucket
          AND dbr.IS_OPTION          = d.IS_OPTION
          AND dbr.NEW_CONVENTION_NAME= d.NEW_CONVENTION_NAME
),
/* -------- 8. построчный слой (Opened) -------------------------------- */
OpenLine AS (
    SELECT
        d.MonthEnd,
        CASE WHEN d.SEG_NAME='Розничный бизнес' THEN N'Розница'
             WHEN d.SEG_NAME='ДЧБО'             THEN N'ЧБО'
             ELSE N'Без сегментации' END      AS SegmentGrouping,
        d.PROD_NAME,
        d.CUR                                 AS CurrencyGrouping,
        d.TermBucket                          AS TermBucketGrouping,
        d.BalanceBucket                       AS BalanceBucketGrouping,
        CAST(d.IS_OPTION AS nvarchar)         AS IS_OPTION,

        'Opened'           AS DealType,
        d.CON_ID,
        d.CLI_ID,
        d.DT_OPEN,
        d.DT_CLOSE_PLAN     AS DT_Matur,
        d.BALANCE_RUB       AS BalanceRub,
        d.ConvertedRate,
        d.BaseRate,
        d.Discount,
        d.ProlongCount,

        /* Int-% (для открытых = NULL) */
        CAST(NULL AS money)                   AS Int_Balance_RUB,

        /* Count-flags */
        CASE WHEN d.ProlongCount>0 THEN 1 ELSE 0 END AS Count_Prolong,
        CASE WHEN d.ProlongCount=0 THEN 1 ELSE 0 END AS Count_NewNoProlong,
        CASE WHEN d.ProlongCount=1 THEN 1 ELSE 0 END AS Count_1yProlong,
        CASE WHEN d.ProlongCount=2 THEN 1 ELSE 0 END AS Count_2yProlong,
        CASE WHEN d.ProlongCount=3 THEN 1 ELSE 0 END AS Count_3yProlong,
        CASE WHEN d.ProlongCount=4 THEN 1 ELSE 0 END AS Count_4yProlong,
        CASE WHEN d.ProlongCount>=5 THEN 1 ELSE 0 END AS Count_5plusProlong,

        /* Sums (руб) */
        CASE WHEN d.ProlongCount>0 THEN d.BALANCE_RUB ELSE 0 END AS Sum_ProlongRub,
        CASE WHEN d.ProlongCount=0 THEN d.BALANCE_RUB ELSE 0 END AS Sum_NewNoProlong,
        CASE WHEN d.ProlongCount=1 THEN d.BALANCE_RUB ELSE 0 END AS Sum_1yProlong_Rub,
        CASE WHEN d.ProlongCount=2 THEN d.BALANCE_RUB ELSE 0 END AS Sum_2yProlong_Rub,
        CASE WHEN d.ProlongCount=3 THEN d.BALANCE_RUB ELSE 0 END AS Sum_3yProlong_Rub,
        CASE WHEN d.ProlongCount=4 THEN d.BALANCE_RUB ELSE 0 END AS Sum_4yProlong_Rub,
        CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_RUB ELSE 0 END AS Sum_5plusProlong_Rub,

        /* весовые ставки */
        d.BALANCE_RUB*d.ConvertedRate                         AS Sum_RateWeighted,
        CASE WHEN d.ProlongCount=0  THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_NewNoProlong,
        CASE WHEN d.ProlongCount=1  THEN d.BALANCE_Rub*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_1y,
        CASE WHEN d.ProlongCount=2  THEN d.BALANCE_Rub*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_2y,
        CASE WHEN d.ProlongCount=3  THEN d.BALANCE_Rub*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_3y,
        CASE WHEN d.ProlongCount=4  THEN d.BALANCE_Rub*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_4y,
        CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_Rub*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_5plus,

        /* веса дисконтов */
        d.BALANCE_Rub*d.Discount                             AS Sum_Discount,
        CASE WHEN d.ProlongCount=0  THEN d.BALANCE_Rub*d.Discount ELSE 0 END AS Sum_Discount_NewNoProlong,
        CASE WHEN d.ProlongCount=1  THEN d.BALANCE_Rub*d.Discount ELSE 0 END AS Sum_Discount_1y,
        CASE WHEN d.ProlongCount=2  THEN d.BALANCE_Rub*d.Discount ELSE 0 END AS Sum_Discount_2y,
        CASE WHEN d.ProlongCount=3  THEN d.BALANCE_Rub*d.Discount ELSE 0 END AS Sum_Discount_3y,
        CASE WHEN d.ProlongCount=4  THEN d.BALANCE_Rub*d.Discount ELSE 0 END AS Sum_Discount_4y,
        CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_Rub*d.Discount ELSE 0 END AS Sum_Discount_5plus,

        /* %-суммы (для opened = NULL) */
        CAST(NULL AS money) AS Sum_ProlongRub_int,
        CAST(NULL AS money) AS Sum_NewNoProlong_int
    FROM DealsWithDiscount d
),
/* ======================================================================= */
/*  B.  З А К Р Ы Т Ы Е                                                    */
/* ======================================================================= */
ClosedDealsInMonthRates AS (
    SELECT
        M.MonthEnd,
        CAST(dc.CON_ID AS bigint)            AS CON_ID,
        dc.SEG_NAME,
        ISNULL(dc.PROD_NAME,'Без типа')      AS PROD_NAME,
        dc.CUR,
        dc.DT_OPEN,
        dc.DT_CLOSE,
        dc.DT_CLOSE_PLAN,
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
                   1
               )*0.9525 - 0.0048)               AS ConvertedRate
    FROM  cteMonths M
    JOIN  LIQUIDITY.liq.DepositContract_all dc WITH (NOLOCK)
           ON dc.DT_CLOSE >= @MonthBeg
          AND dc.DT_CLOSE <  DATEADD(DAY,1,@MonthEnd)
          AND dc.CLI_SUBTYPE = 'INDIV'
          AND dc.PROD_NAME  <> 'Эскроу'
          AND dc.CONVENTION <> 'ON_DEMAND'
          AND dc.DT_CLOSE_PLAN <> '4444-01-01'
          AND DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE) > 10
    JOIN  LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
           ON conv.CONVENTION_NAME = dc.CONVENTION
    LEFT JOIN ALM_TEST.WORK.DepositInterestsRateSnap snap WITH (NOLOCK)
           ON snap.CON_ID = dc.CON_ID
    WHERE CAST(dc.CON_ID AS bigint) NOT IN
          (SELECT CON_ID FROM LIQUIDITY.liq.man_FloatContracts WITH (NOLOCK))
),
ClosedDealsInMonthBucket AS (
    SELECT  cdm.*,
            tg.TERM_GROUP       AS TermBucket,
            bal.BALANCE_GROUP   AS BalanceBucket,
            CAST(ISNULL(opt.IS_OPTION,0) AS nvarchar) AS IS_OPTION,
            /* начисленные % (лишь для RUR, non-option) */
            CASE WHEN cdm.CUR='RUR' AND ISNULL(opt.IS_OPTION,0)=0
                 THEN CASE WHEN cdm.MATUR=cdm.DaysLived
                           THEN cdm.BALANCE_RUB*(POWER(1+cdm.ConvertedRate/12,
                                                      cdm.DaysLived/31)-1)
                           ELSE cdm.BALANCE_RUB*(POWER(1+0.01/1200,
                                                      cdm.DaysLived/31)-1)
                      END
                 ELSE 0 END  AS Int_Balance_RUB
    FROM ClosedDealsInMonthRates cdm
    LEFT JOIN ALM_TEST.WORK.man_TermGroup    tg  WITH (NOLOCK)
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
    FROM  ClosedDealsInMonthBucket dmb
    LEFT JOIN ClosedChainDepth ccd ON ccd.StartConId = dmb.CON_ID
),
ClosedLine AS (
    SELECT
        d.MonthEnd,
        CASE WHEN d.SEG_NAME='Розничный бизнес' THEN N'Розница'
             WHEN d.SEG_NAME='ДЧБО'             THEN N'ЧБО'
             ELSE N'Без сегментации' END      AS SegmentGrouping,
        d.PROD_NAME,
        d.CUR                                 AS CurrencyGrouping,
        d.TermBucket                          AS TermBucketGrouping,
        d.BalanceBucket                       AS BalanceBucketGrouping,
        CAST(d.IS_OPTION AS nvarchar)         AS IS_OPTION,

        'Closed'          AS DealType,
        d.CON_ID,
        NULL              AS CLI_ID,
        d.DT_OPEN,
        d.DT_CLOSE        AS DT_Matur,
        d.BALANCE_RUB     AS BalanceRub,
        d.ConvertedRate,
        NULL              AS BaseRate,
        NULL              AS Discount,
        d.ProlongCount,
        d.Int_Balance_RUB AS Int_Balance_RUB,

        /* Count-flags (без префикса) */
        CASE WHEN d.ProlongCount>0 THEN 1 ELSE 0 END AS Count_Prolong,
        CASE WHEN d.ProlongCount=0 THEN 1 ELSE 0 END AS Count_NewNoProlong,
        CASE WHEN d.ProlongCount=1 THEN 1 ELSE 0 END AS Count_1yProlong,
        CASE WHEN d.ProlongCount=2 THEN 1 ELSE 0 END AS Count_2yProlong,
        CASE WHEN d.ProlongCount=3 THEN 1 ELSE 0 END AS Count_3yProlong,
        CASE WHEN d.ProlongCount=4 THEN 1 ELSE 0 END AS Count_4yProlong,
        CASE WHEN d.ProlongCount>=5 THEN 1 ELSE 0 END AS Count_5plusProlong,

        /* Sums (руб) */
        CASE WHEN d.ProlongCount>0 THEN d.BALANCE_RUB ELSE 0 END AS Sum_ProlongRub,
        CASE WHEN d.ProlongCount=0 THEN d.BALANCE_RUB ELSE 0 END AS Sum_NewNoProlong,
        CASE WHEN d.ProlongCount=1 THEN d.BALANCE_RUB ELSE 0 END AS Sum_1yProlong_Rub,
        CASE WHEN d.ProlongCount=2 THEN d.BALANCE_RUB ELSE 0 END AS Sum_2yProlong_Rub,
        CASE WHEN d.ProlongCount=3 THEN d.BALANCE_RUB ELSE 0 END AS Sum_3yProlong_Rub,
        CASE WHEN d.ProlongCount=4 THEN d.BALANCE_RUB ELSE 0 END AS Sum_4yProlong_Rub,
        CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_RUB ELSE 0 END AS Sum_5plusProlong_Rub,

        /* ставки — закрытому уже не нужны (0) */
        0 AS Sum_RateWeighted,
        0 AS Sum_RateWeighted_NewNoProlong,
        0 AS Sum_RateWeighted_1y,
        0 AS Sum_RateWeighted_2y,
        0 AS Sum_RateWeighted_3y,
        0 AS Sum_RateWeighted_4y,
        0 AS Sum_RateWeighted_5plus,

        /* дисконты — не нужны */
        0 AS Sum_Discount,
        0 AS Sum_Discount_NewNoProlong,
        0 AS Sum_Discount_1y,
        0 AS Sum_Discount_2y,
        0 AS Sum_Discount_3y,
        0 AS Sum_Discount_4y,
        0 AS Sum_Discount_5plus,

        /* %-суммы */
        CASE WHEN d.ProlongCount>0 THEN d.Int_Balance_RUB ELSE 0 END AS Sum_ProlongRub_int,
        CASE WHEN d.ProlongCount=0 THEN d.Int_Balance_RUB ELSE 0 END AS Sum_NewNoProlong_int
)
/* ===================================================================== */
/*  C.  O U T P U T                                                      */
/* ===================================================================== */
SELECT *
INTO   #LineDump202504           -- уберите INTO, если нужен прямой SELECT
FROM   OpenLine
UNION ALL
SELECT * FROM ClosedLine;

/* Проверка */
SELECT TOP (100) *
FROM   #LineDump202504
ORDER BY DealType, PROD_NAME, CON_ID;
