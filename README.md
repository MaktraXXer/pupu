/* Работает в каждой среде, но без защиты от случайного дропа представления */
USE ALM_TEST;
GO

CREATE OR ALTER FUNCTION WORK.ufn_GetDepositForecastRates
(
    @OpenDate  DATE,
    @Term      INT
)
RETURNS TABLE
AS
RETURN
(
    /* Определяем самую свежую выгрузку */
    WITH LastRep AS (
        SELECT MAX(DT_REP) AS LastRep
        FROM  ALM.info.VW_ForecastKEY_everyday WITH (NOLOCK)
        WHERE DT_REP <= CAST(GETDATE() AS DATE)
    ),
    Params AS (
        SELECT 
            @OpenDate                       AS OpenDate ,
            DATEADD(day,@Term,@OpenDate)    AS CloseDate ,
            @Term                           AS Term
    )
    SELECT
        open_fk.AVG_KEY_RATE AS AvgRateAtOpen,
        ( SELECT AVG(fk.KEY_RATE)
          FROM  ALM.info.VW_ForecastKEY_everyday fk WITH (NOLOCK)
          CROSS JOIN LastRep lr
          CROSS JOIN Params  p
          WHERE fk.DT_REP = lr.LastRep
            AND fk.[Date] >= p.CloseDate
            AND fk.[Date] <  DATEADD(day,p.Term,p.CloseDate)
        )                     AS AvgRateAtClose
    FROM  ALM.info.VW_ForecastKEY_everyday open_fk WITH (NOLOCK)
    CROSS JOIN Params p
    WHERE open_fk.DT_REP = p.OpenDate
      AND open_fk.Term   = p.Term
);
GO
