DECLARE @DateTo date = CAST(GETDATE() AS date);
DECLARE @DateFrom date = DATEADD(month, -6, @DateTo);

;WITH base AS (

    SELECT
        cli_id,

        ISNULL([Распознано_СР], 0)
        + ISNULL([Распознано_НС], 0)
        + ISNULL([Распознано_ДВС], 0) AS amt,

        ISNULL(count_in, 0) AS count_in,
        ISNULL(count_out, 0) AS count_out

    FROM ALM.ehd.VW_transfers_FL_det

    WHERE dt_rep >= @DateFrom
      AND dt_rep < @DateTo
      AND transaction_type = N'Наличные'
      AND cli_id IS NOT NULL

)

SELECT
    cli_id,

    /* Флаги */

    MAX(CASE WHEN amt < 0
             THEN 1 ELSE 0 END) AS cash_out_flag,

    MAX(CASE WHEN amt > 0
             THEN 1 ELSE 0 END) AS cash_in_flag,

    MAX(CASE WHEN amt <= -1000000
             THEN 1 ELSE 0 END) AS cash_out_1m_flag,

    MAX(CASE WHEN amt >= 1000000
             THEN 1 ELSE 0 END) AS cash_in_1m_flag,

    /* Распознанные объемы */

    SUM(CASE WHEN amt > 0
             THEN amt ELSE 0 END) AS cash_in,

    SUM(CASE WHEN amt < 0
             THEN amt ELSE 0 END) AS cash_out,

    SUM(CASE WHEN amt >= 1000000
             THEN amt ELSE 0 END) AS cash_in_1m,

    SUM(CASE WHEN amt <= -1000000
             THEN amt ELSE 0 END) AS cash_out_1m,

    SUM(amt) AS cash_net,

    /* Количество операций по соответствующим строкам */

    SUM(CASE WHEN amt > 0
             THEN count_in ELSE 0 END) AS count_in,

    SUM(CASE WHEN amt < 0
             THEN count_out ELSE 0 END) AS count_out

FROM base

WHERE amt <> 0

GROUP BY cli_id

HAVING MAX(ABS(amt)) >= 1000000

OPTION (MAXDOP 1, RECOMPILE);
