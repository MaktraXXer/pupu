USE ALM_TEST;
GO

CREATE OR ALTER FUNCTION WORK.ufn_GetDepositForecastRates
(
    @OpenDate DATE,   -- дата открытия «старого» депозита
    @Term     INT     -- срочность, дней
)
RETURNS TABLE
AS
RETURN
/* ----------------------------------------------------------
   1. Подставляем параметры
---------------------------------------------------------- */
WITH P AS (
    SELECT  @OpenDate                           AS OpenDate ,
            DATEADD(day,@Term,@OpenDate)        AS CloseDate ,
            @Term                               AS Term
),
/* ----------------------------------------------------------
   2. Находим «активный» снимок прогноза на дату открытия
      (тот же алгоритм, что во витрине: DT_REP_NEXT > OpenDate)
---------------------------------------------------------- */
OpenRep AS (
    SELECT TOP (1) fk.DT_REP
    FROM (
        SELECT  DT_REP ,
                LEAD(DT_REP) OVER (ORDER BY DT_REP) AS DT_REP_NEXT
        FROM    LIQUIDITY.liq.ForecastKeyRate
        GROUP  BY DT_REP          -- одна строка на снимок
    ) fk
    CROSS JOIN P
    WHERE fk.DT_REP      <= P.OpenDate
      AND P.OpenDate      < ISNULL(fk.DT_REP_NEXT,'30000101')
    ORDER BY fk.DT_REP DESC          -- берём самый свежий
),
/* ----------------------------------------------------------
   3. Самый свежий прогноз на сегодня
---------------------------------------------------------- */
LastRep AS (
    SELECT MAX(DT_REP) AS DT_REP
    FROM   LIQUIDITY.liq.ForecastKeyRate
    WHERE  DT_REP <= CAST(GETDATE() AS DATE)
)
/* ----------------------------------------------------------
   4. Итог одной строкой
---------------------------------------------------------- */
SELECT
    /* --- средняя ставка, которую клиент "видел" при открытии --- */
    ( SELECT AVG(KEY_RATE)
      FROM   LIQUIDITY.liq.ForecastKeyRate fk
             CROSS JOIN OpenRep  o
             CROSS JOIN P
      WHERE  fk.DT_REP = o.DT_REP
        AND  fk.[Date] BETWEEN P.OpenDate                 -- включительно
                           AND DATEADD(day,P.Term-1,P.OpenDate)
    ) AS AvgRateAtOpen ,

    /* --- средняя ставка для нового депозита (снимок LastRep) --- */
    ( SELECT AVG(KEY_RATE)
      FROM   LIQUIDITY.liq.ForecastKeyRate fk
             CROSS JOIN LastRep lr
             CROSS JOIN P
      WHERE  fk.DT_REP = lr.DT_REP
        AND  fk.[Date] BETWEEN P.CloseDate
                           AND DATEADD(day,P.Term-1,P.CloseDate)
    ) AS AvgRateAtClose ;
GO

DECLARE @OpenDate DATE = '2025-04-20',
        @Term     INT  = 91;

SET STATISTICS TIME ON;
SELECT * 
FROM   WORK.ufn_GetDepositForecastRates(@OpenDate,@Term);
SET STATISTICS TIME OFF;
