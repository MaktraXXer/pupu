DECLARE @DateTo date = CAST(GETDATE() AS date);
DECLARE @DateFrom date = DATEADD(month, -6, @DateTo);

SELECT
    cli_id,

    /* Флаги */

    MAX(CASE WHEN amount_out < 0
             THEN 1 ELSE 0 END) AS cash_out_flag,

    MAX(CASE WHEN amount_in > 0
             THEN 1 ELSE 0 END) AS cash_in_flag,

    MAX(CASE WHEN amount_out <= -1000000
             THEN 1 ELSE 0 END) AS cash_out_1m_flag,

    MAX(CASE WHEN amount_in >= 1000000
             THEN 1 ELSE 0 END) AS cash_in_1m_flag,

    /* Объемы */

    SUM(ISNULL(amount_in, 0)) AS cash_in,

    SUM(ISNULL(amount_out, 0)) AS cash_out,

    SUM(CASE WHEN amount_in >= 1000000
             THEN amount_in ELSE 0 END) AS cash_in_1m,

    SUM(CASE WHEN amount_out <= -1000000
             THEN amount_out ELSE 0 END) AS cash_out_1m,

    /* Количество операций */

    SUM(ISNULL(count_in, 0)) AS count_in,

    SUM(ISNULL(count_out, 0)) AS count_out

FROM ALM.ehd.VW_transfers_FL_det

WHERE dt_rep >= @DateFrom
  AND dt_rep < @DateTo
  AND transaction_type = N'Наличные'
  AND cli_id IS NOT NULL
  AND (amount_in > 0 OR amount_out < 0)

GROUP BY cli_id

OPTION (MAXDOP 1, RECOMPILE);
