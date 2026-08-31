USE [LIQUIDITY];
SET NOCOUNT ON;

DECLARE @CliId bigint = 1234567890;
DECLARE @DateFrom date = '2026-05-31';
DECLARE @DateTo   date = '2026-08-31';


/* ============================================================
   Договоры конкретного клиента + история их фактического сальдо
   ============================================================ */

WITH contracts AS
(
    SELECT
          d.CON_ID
        , d.CLI_ID
        , MIN(CAST(d.DT_OPEN AS date)) AS DT_OPEN
        , MAX(CAST(d.DT_CLOSE_PLAN AS date)) AS DT_CLOSE_PLAN
        , MAX(d.PROD_TYPE) AS PROD_TYPE
        , MAX(d.PROD_NAME) AS PROD_NAME
        , MAX(d.TSEGMENTNAME) AS TSEGMENTNAME
        , MAX(d.isfloat) AS isfloat

    FROM [LIQUIDITY].[liq].[DepositInterestRate] d WITH (NOLOCK)

    WHERE
        d.CLI_ID = @CliId

    GROUP BY
          d.CON_ID
        , d.CLI_ID
)

SELECT
      c.CLI_ID
    , c.CON_ID

    , c.DT_OPEN
    , c.DT_CLOSE_PLAN

    , c.PROD_TYPE
    , c.PROD_NAME
    , c.TSEGMENTNAME
    , c.isfloat

    /* период, в котором действовал этот остаток */
    , CAST(s.DT_FROM AS date) AS saldo_from
    , CAST(s.DT_TO AS date) AS saldo_to

    /* фактический остаток */
    , CAST(s.OUT_RUB AS decimal(38,2)) AS out_rub

FROM contracts c

INNER JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] s WITH (NOLOCK)
    ON s.CON_ID = c.CON_ID

WHERE
    s.DT_FROM <= @DateTo

    AND
    (
        s.DT_TO IS NULL
        OR s.DT_TO >= @DateFrom
    )

ORDER BY
      c.CON_ID
    , s.DT_FROM;
