/*=====================================================================
  LineDump (апрель-2025): построчные данные открытых / закрытых ФЛ
  Полная логика = витрине WORK.prolongationAnalysisResult_ISOPTION
=====================================================================*/
USE ALM_TEST;
GO
SET NOCOUNT ON;
SET ANSI_NULLS , QUOTED_IDENTIFIER ON;
GO

/* ---------- параметры периода ---------- */
DECLARE @MonthEnd  date = '2025-04-30';
DECLARE @MonthBeg  date = DATEADD(DAY,1,EOMONTH(@MonthEnd,-1));
DECLARE @DayWindow int  = 3;        -- ±N дней для поиска базы

/* ====================================================================
   Часть A :  О Т К Р Ы Т Ы Е   Д Е П О З И Т Ы
   ================================================================== */
WITH
/* ---------- 0. признак опциональности -------------------------------*/
ISOPT AS (
    SELECT CAST(CON_ID AS bigint)                        AS CON_ID,
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
        dc.DT_CLOSE_PLAN      AS DT_CLOSE_PLAN,          -- ← ДОБАВЛЕНО!
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

/* ---------- 3. бакеты срочности / сумм ------------------------------ */
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

/* ---------- 5. первый «старый» вклад ------------------------------- */
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

/* ---------- 6. объединяем ------------------------------------------ */
DealsWithProlong AS (
    SELECT dmb.*,
           ISNULL(cd.ProlongCount,0)              AS ProlongCount,
           ISNULL(pi.PrevBalance,0)               AS PrevBalance,
           ISNULL(pi.PrevConvertedRate,0)         AS PrevConvertedRate
    FROM  DealsInMonthBucket dmb
    LEFT JOIN ChainDepth cd ON cd.StartConId  = dmb.CON_ID
    LEFT JOIN PrevInfo  pi ON pi.StartConId   = dmb.CON_ID
),

/* ---------- 7. логика базовой ставки / дисконта (как в витрине) ----- */
-- … (вся логика BaseCandidates / BestBase / DealsWithDiscount остаётся без изменений)

-- <<< здесь идёт длинный участок «BalanceBuckets / BaseCandidates / … / DealsWithDiscount»
--     – он НЕ менялся, поэтому опущен ради компактности. >>>

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
        d.DT_CLOSE_PLAN AS DT_Matur,            -- плановая (!) дата закрытия
        d.BALANCE_RUB   AS BalanceRub,
        d.ConvertedRate,
        d.BaseRate,
        d.Discount,
        d.ProlongCount,
        /* … остальные поля без изменений … */
        -- (блок Count_…  Sum_…  и т.д.)
        -- -----------------------------------
        CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_5plus
    FROM DealsWithDiscount d
),

/* ====================================================================
   Часть B :  З А К Р Ы Т Ы Е
   ================================================================== */
-- (весь кусок с ClosedDealsInMonthRates → ClosedLine без изменений)

-- …

/* ====================================================================
   Финал: opened ∪ closed
=====================================================================*/
SELECT *
INTO   #LineDump202504         -- временная таблица (уберите INTO если не нужна)
FROM   OpenLine
UNION ALL
SELECT *
FROM   ClosedLine;

SELECT TOP (100) *
FROM   #LineDump202504
ORDER  BY DealType, PROD_NAME, CON_ID;
