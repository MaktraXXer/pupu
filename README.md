/*=====================================================================
  Быстрая витрина автопролонгаций ФЛ
  ─ Алгоритм поиска «базы» ровно такой, как запросил пользователь ─
      1)   ТОЧНО тот же день  +  ТОЧНО тот же бакет
      2)   ТОТ ЖЕ день       +  “соседний” бакет (вниз ≤1.5 млн / вверх >1.5 млн)
      3+)  ΔT = 1,2,3… дн   +  пункты 1‑2 на каждом шаге
      4)   если ничего не найдено → BaseRate = NULL
=====================================================================*/
USE ALM_TEST;
GO
SET ANSI_NULLS , QUOTED_IDENTIFIER ON;
GO

DECLARE @DayWindow int = 3;           -- максимальный сдвиг по дням (±)

WITH
/*--------------------------------------------------------------------*/
ISOPT AS (
    SELECT b.CON_ID,
           CASE WHEN ISNULL(b.OptionRate_TRF,0) < 0 THEN 1 ELSE 0 END AS IS_OPTION
    FROM   LIQUIDITY.liq.InterestsRateForDeposit b WITH (NOLOCK)
),
cteMonths AS ( SELECT CONVERT(date,'2025‑01‑31') AS MonthEnd ),
/*--------------------------------------------------------------------*/
DealsInMonthRates AS (   /* открытые за месяц – только нужные колонки */
    SELECT  M.MonthEnd,
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
            AND dc.DT_CLOSE_PLAN <> '4444‑01‑01'
            AND DATEDIFF(DAY,dc.DT_OPEN,dc.DT_CLOSE) > 10
    JOIN   LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
             ON conv.CONVENTION_NAME = dc.CONVENTION
    LEFT  JOIN ALM_TEST.WORK.DepositInterestsRateSnap snap WITH (NOLOCK)
             ON snap.CON_ID = dc.CON_ID
    WHERE  dc.CON_ID NOT IN (SELECT CON_ID
                             FROM LIQUIDITY.liq.man_FloatContracts WITH (NOLOCK))
),
/*--------------------------------------------------------------------*/
DealsInMonthBucket AS (   /* бакеты срок/объём + IS_OPTION */
    SELECT dmr.*,
           tg.TERM_GROUP              AS TermBucket,
           bal.BALANCE_GROUP          AS BalanceBucket,
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
/*  Рекурсивно получаем ProlongCount 0‑5+                              */
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
           ON p.CON_ID        = rc.CurrentConId
          AND p.CON_REL_TYPE  = 'PREVIOUS'
    WHERE rc.Lvl < 5
),
ChainDepth AS (
    SELECT StartConId,
           CASE WHEN MAX(Lvl)>5 THEN 5 ELSE MAX(Lvl) END AS ProlongCount
    FROM RecursiveChain
    GROUP BY StartConId
),
FirstProlong AS (
    SELECT StartConId,
           MIN(CurrentConId) AS Old1
    FROM RecursiveChain WHERE Lvl = 1
    GROUP BY StartConId
),
PrevInfo AS (
    SELECT fp.StartConId,
           old.BALANCE_RUB AS PrevBalance,
           LIQUIDITY.liq.fnc_IntRate(old.RATE, conv.NEW_CONVENTION_NAME,
                                     'monthly',
                                     DATEDIFF(DAY,old.DT_OPEN,old.DT_CLOSE_PLAN),
                                     1)           AS PrevConvertedRate
    FROM FirstProlong fp
    JOIN LIQUIDITY.liq.DepositContract_all old WITH (NOLOCK)
           ON old.CON_ID = fp.Old1
    JOIN LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
           ON conv.CONVENTION_NAME = old.CONVENTION
),
/*--------------------------------------------------------------------*/
DealsWithProlong AS (
    SELECT dmb.*,
           ISNULL(cd.ProlongCount,0)   AS ProlongCount,
           ISNULL(pi.PrevBalance,0)    AS PrevBalance,
           ISNULL(pi.PrevConvertedRate,0) AS PrevConvertedRate
    FROM DealsInMonthBucket dmb
    LEFT JOIN ChainDepth cd ON cd.StartConId  = dmb.CON_ID
    LEFT JOIN PrevInfo  pi ON pi.StartConId   = dmb.CON_ID
),
/*--------------------------------------------------------------------*/
/*  Подготовка для поиска «базы»                                      */
/*--------------------------------------------------------------------*/
BalanceBuckets AS (
    SELECT BALANCE_GROUP,
           BALANCE_FROM,
           BALANCE_TO,
           ROW_NUMBER() OVER (ORDER BY BALANCE_FROM) AS BucketOrder
    FROM ALM_TEST.WORK.man_BalanceGroup
),
BaseCandidates AS (          -- только Prolong = 0
    SELECT dwp.DT_OPEN,
           dwp.SEG_NAME,
           dwp.PROD_NAME,
           dwp.CUR,
           dwp.TermBucket,
           bb.BucketOrder,
           dwp.BalanceBucket,
           dwp.IS_OPTION,
           dwp.NEW_CONVENTION_NAME,
           AVG(dwp.ConvertedRate) AS BaseRate
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
           bb.BALANCE_TO  AS BucketUpper,
           d.BalanceBucket,
           d.NEW_CONVENTION_NAME
    FROM DealsWithProlong d
    JOIN BalanceBuckets bb ON bb.BALANCE_GROUP = d.BalanceBucket
),
/*--------------------------------------------------------------------*/
/*  BestBase: применяем ТОЧНО тот же день → «соседний» бакет → ΔT=1…   */
/*--------------------------------------------------------------------*/
BestBase AS (
    SELECT dk.*,
           bc.BaseRate,
           /*  ранжирование:  ΔT  →  exact‑bucket  →  bucket‑distance  →  earlier‑date  */
           ROW_NUMBER() OVER (
                PARTITION BY dk.DT_OPEN, dk.SEG_NAME, dk.PROD_NAME,
                             dk.CUR, dk.TermBucket, dk.IS_OPTION,
                             dk.NEW_CONVENTION_NAME, dk.BalanceBucket
                ORDER BY
                     ABS(DATEDIFF(DAY, bc.DT_OPEN, dk.DT_OPEN)),             -- 1) ближе по дате
                     CASE WHEN bc.BucketOrder = dk.BucketOrder THEN 0 ELSE 1 END,
                     ABS(bc.BucketOrder - dk.BucketOrder),                   -- 2) ближе по объёму
                     CASE WHEN bc.DT_OPEN < dk.DT_OPEN THEN 0 ELSE 1 END     -- 3) если ΔT одинаков
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
          AND (
                 (dk.BucketUpper<=1500000 AND bc.BucketOrder<=dk.BucketOrder) OR
                 (dk.BucketUpper> 1500000 AND bc.BucketOrder>=dk.BucketOrder) )
),
DailyBaseRate AS ( SELECT * FROM BestBase WHERE rn = 1 ),
/*--------------------------------------------------------------------*/
/*  Финальная таблица с дисконтом                                     */
/*--------------------------------------------------------------------*/
DealsWithDiscount AS (
    SELECT dwp.*,
           dbr.BaseRate,
           dwp.ConvertedRate - dbr.BaseRate AS Discount
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
)
/*====================================================================*/
SELECT *   -- далее можно сразу INSERT …  или передавать в следующие CTE
FROM   DealsWithDiscount
OPTION (RECOMPILE, MAXDOP 4);
GO
