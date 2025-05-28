/* ---------------------------------------------------------
   LineDump: opened + closed INDIV deposits (апрель-2025)
   Поля подготовлены так же, как в OpenAggregated_0 / ClosedAggregated_0
   ------------------------------------------------------- */
USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @MonthEnd  date = '2025-04-30';
DECLARE @MonthBeg  date = DATEADD(DAY,1,EOMONTH(@MonthEnd,-1));
DECLARE @DayWindow int  = 3;       --  ±N дней для поиска базы (см. оригинальный скрипт)

/* ---------- справочник «опционности» ------------------- */
WITH ISOPT AS (
    SELECT  CAST(CON_ID AS bigint) CON_ID,
            CASE WHEN ISNULL(OptionRate_TRF,0)<0 THEN 1 ELSE 0 END AS IS_OPTION
    FROM    LIQUIDITY.liq.InterestsRateForDeposit  (NOLOCK)
),

/* --------------------------------------------------------
   -------   О Т К Р Ы Т Ы Е   ----------------------------
   (до DealsWithDiscount включительно – без агрегации)
   ------------------------------------------------------ */
DealsOpen AS (
    /* весь кусок до DealsWithDiscount из вашего скрипта,
       НИЧЕГО не меняем, только оставляем нужные поля      */
    /* ………………………………………… */
    /* вместо  - показываю только последний CTE ---------- */
    SELECT  dwp.*,
            /* правило BaseRate + Discount */
            CASE WHEN dwp.ProlongCount = 0
                 THEN dwp.ConvertedRate
                 ELSE dbr.BaseRate END                       AS BaseRate,
            CASE WHEN dwp.ProlongCount = 0
                 THEN 0
                 WHEN dbr.BaseRate IS NULL
                 THEN NULL
                 ELSE dwp.ConvertedRate - dbr.BaseRate END   AS Discount
    FROM    DealsWithProlong dwp
    LEFT JOIN DailyBaseRate dbr
           ON  dbr.DT_OPEN             = dwp.DT_OPEN
          AND dbr.SEG_NAME             = dwp.SEG_NAME
          AND dbr.PROD_NAME            = dwp.PROD_NAME
          AND dbr.CUR                  = dwp.CUR
          AND dbr.TermBucket           = dwp.TermBucket
          AND dbr.BalanceBucket        = dwp.BalanceBucket
          AND dbr.IS_OPTION            = dwp.IS_OPTION
          AND dbr.NEW_CONVENTION_NAME  = dwp.NEW_CONVENTION_NAME
),

/* построчные «агрегатные» флаги/суммы ------------------- */
OpenLine AS (
SELECT
    /* -- ключи группировки -- */
    d.MonthEnd,
    CASE WHEN d.SEG_NAME='Розничный бизнес' THEN N'Розница'
         WHEN d.SEG_NAME='ДЧБО'             THEN N'ЧБО'
         ELSE N'Без сегментации' END            AS SegmentGrouping,
    d.PROD_NAME,
    d.CUR                                       AS CurrencyGrouping,
    d.TermBucket                                AS TermBucketGrouping,
    d.BalanceBucket                             AS BalanceBucketGrouping,
    CAST(d.IS_OPTION AS nvarchar)               AS IS_OPTION,

    /* --- служебные --- */
    'Opened'            AS DealType,
    d.CON_ID,
    d.CLI_ID,
    d.DT_OPEN,
    d.DT_CLOSE_PLAN     AS DT_Matur,
    d.BALANCE_RUB       AS BalanceRub,
    d.ConvertedRate,
    d.BaseRate,
    d.Discount,
    d.ProlongCount,

    /* ---- «Count_…» ---- */
    CASE WHEN d.ProlongCount>0 THEN 1 ELSE 0 END AS Count_Prolong,
    CASE WHEN d.ProlongCount=0 THEN 1 ELSE 0 END AS Count_NewNoProlong,
    CASE WHEN d.ProlongCount=1 THEN 1 ELSE 0 END AS Count_1yProlong,
    CASE WHEN d.ProlongCount=2 THEN 1 ELSE 0 END AS Count_2yProlong,
    CASE WHEN d.ProlongCount=3 THEN 1 ELSE 0 END AS Count_3yProlong,
    CASE WHEN d.ProlongCount=4 THEN 1 ELSE 0 END AS Count_4yProlong,
    CASE WHEN d.ProlongCount>=5 THEN 1 ELSE 0 END AS Count_5plusProlong,

    /* ---- «Sum_…»  (0 либо сумма договора) ---- */
    CASE WHEN d.ProlongCount>0 THEN d.BALANCE_RUB ELSE 0 END AS Sum_ProlongRub,
    CASE WHEN d.ProlongCount=0 THEN d.BALANCE_RUB ELSE 0 END AS Sum_NewNoProlong,
    CASE WHEN d.ProlongCount=1 THEN d.BALANCE_RUB ELSE 0 END AS Sum_1yProlong_Rub,
    CASE WHEN d.ProlongCount=2 THEN d.BALANCE_RUB ELSE 0 END AS Sum_2yProlong_Rub,
    CASE WHEN d.ProlongCount=3 THEN d.BALANCE_RUB ELSE 0 END AS Sum_3yProlong_Rub,
    CASE WHEN d.ProlongCount=4 THEN d.BALANCE_RUB ELSE 0 END AS Sum_4yProlong_Rub,
    CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_RUB ELSE 0 END AS Sum_5plusProlong_Rub,

    /* взвешенные ставки / дисконты – то же самое */
    d.BALANCE_RUB * d.ConvertedRate                    AS Sum_RateWeighted,
    CASE WHEN d.ProlongCount=0  THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_NewNoProlong,
    CASE WHEN d.ProlongCount=1  THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_1y,
    CASE WHEN d.ProlongCount=2  THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_2y,
    CASE WHEN d.ProlongCount=3  THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_3y,
    CASE WHEN d.ProlongCount=4  THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_4y,
    CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_RUB*d.ConvertedRate ELSE 0 END AS Sum_RateWeighted_5plus,

    d.BALANCE_RUB * d.Discount                        AS Sum_Discount,
    CASE WHEN d.ProlongCount=0  THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_NewNoProlong,
    CASE WHEN d.ProlongCount=1  THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_1y,
    CASE WHEN d.ProlongCount=2  THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_2y,
    CASE WHEN d.ProlongCount=3  THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_3y,
    CASE WHEN d.ProlongCount=4  THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_4y,
    CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_RUB*d.Discount ELSE 0 END AS Sum_Discount_5plus
FROM DealsOpen d
),

/* --------------------------------------------------------
   -------   З А К Р Ы Т Ы Е   ----------------------------
   (до ClosedDealsWithProlong включительно – без агрегации)
   ------------------------------------------------------ */
DealsClosed AS (
    /* берём CTE ClosedDealsWithProlong из вашего скрипта */
    /* ………………………………………… */
    SELECT * FROM ClosedDealsWithProlong
),

ClosedLine AS (
SELECT
    /* -- ключи группировки -- */
    d.MonthEnd,
    CASE WHEN d.SEG_NAME='Розничный бизнес' THEN N'Розница'
         WHEN d.SEG_NAME='ДЧБО'             THEN N'ЧБО'
         ELSE N'Без сегментации' END            AS SegmentGrouping,
    d.PROD_NAME,
    d.CUR                                       AS CurrencyGrouping,
    d.TermBucket                                AS TermBucketGrouping,
    d.BalanceBucket                             AS BalanceBucketGrouping,
    CAST(d.IS_OPTION AS nvarchar)               AS IS_OPTION,

    /* --- служебные --- */
    'Closed'            AS DealType,
    d.CON_ID,
    NULL                AS CLI_ID,
    d.DT_OPEN,
    d.DT_CLOSE,
    d.BALANCE_RUB       AS BalanceRub,
    d.ConvertedRate,
    NULL                AS BaseRate,
    NULL                AS Discount,
    d.ProlongCount,

    /* Count- / Sum- поля по тому же правилу */
    CASE WHEN d.ProlongCount>0 THEN 1 ELSE 0 END AS Closed_Count_Prolong,
    CASE WHEN d.ProlongCount=0 THEN 1 ELSE 0 END AS Closed_Count_NewNoProlong,
    CASE WHEN d.ProlongCount=1 THEN 1 ELSE 0 END AS Closed_Count_1yProlong,
    CASE WHEN d.ProlongCount=2 THEN 1 ELSE 0 END AS Closed_Count_2yProlong,
    CASE WHEN d.ProlongCount=3 THEN 1 ELSE 0 END AS Closed_Count_3yProlong,
    CASE WHEN d.ProlongCount=4 THEN 1 ELSE 0 END AS Closed_Count_4yProlong,
    CASE WHEN d.ProlongCount>=5 THEN 1 ELSE 0 END AS Closed_Count_5plusProlong,

    CASE WHEN d.ProlongCount>0 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_ProlongRub,
    CASE WHEN d.ProlongCount=0 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_NewNoProlong,
    CASE WHEN d.ProlongCount=1 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_1yProlong_Rub,
    CASE WHEN d.ProlongCount=2 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_2yProlong_Rub,
    CASE WHEN d.ProlongCount=3 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_3yProlong_Rub,
    CASE WHEN d.ProlongCount=4 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_4yProlong_Rub,
    CASE WHEN d.ProlongCount>=5 THEN d.BALANCE_RUB ELSE 0 END AS Closed_Sum_5plusProlong_Rub,

    /* проценты */
    d.Int_Balance_RUB                                    AS Int_Balance_RUB,
    CASE WHEN d.ProlongCount>0 THEN d.Int_Balance_RUB ELSE 0 END AS Closed_Sum_ProlongRub_int,
    CASE WHEN d.ProlongCount=0 THEN d.Int_Balance_RUB ELSE 0 END AS Closed_Sum_NewNoProlong_int
    /* … аналитические поля можно продолжать аналогично … */
FROM DealsClosed d
)

/* ------------------- Итоговая выборка ------------------- */
SELECT * 
FROM   OpenLine
UNION ALL
SELECT * 
FROM   ClosedLine
ORDER BY DealType, PROD_NAME, CON_ID;
