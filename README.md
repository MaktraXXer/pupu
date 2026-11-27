DECLARE @DateTo date = CAST(GETDATE() AS date);
DECLARE @DateFrom date = DATEADD(month, -6, @DateTo);

SELECT
    p.cli_id,

    MAX(CASE WHEN v.amt >= 1000000
             THEN 1 ELSE 0 END) AS cash_in_flag,

    MAX(CASE WHEN v.amt <= -1000000
             THEN 1 ELSE 0 END) AS cash_out_flag

FROM ALM.ehd.VW_transfers_FL_det p

CROSS APPLY (
    VALUES (
        ISNULL(p.[Распознано_СР], 0)
      + ISNULL(p.[Распознано_НС], 0)
      + ISNULL(p.[Распознано_ДВС], 0)
    )
) v(amt)

WHERE p.dt_rep >= @DateFrom
  AND p.dt_rep < @DateTo
  AND p.transaction_type = N'Наличные'
  AND p.cli_id IS NOT NULL

  AND (
      (v.amt >= 1000000
       AND p.count_in = 1
       AND ISNULL(p.count_out, 0) = 0)

      OR

      (v.amt <= -1000000
       AND p.count_out = 1
       AND ISNULL(p.count_in, 0) = 0)
  )

GROUP BY p.cli_id

OPTION (MAXDOP 1, RECOMPILE);
