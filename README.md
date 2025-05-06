,DealsWithDiscount AS (                     -- добавляем дисконт
    SELECT dwp.*,
           dbr.BaseRate,
           (dbr.BaseRate - dwp.ConvertedRate)                    AS DiscountAbs,
           CASE WHEN dbr.BaseRate IS NOT NULL AND dbr.BaseRate <> 0
                THEN (dwp.ConvertedRate / dbr.BaseRate) - 1 END AS DiscountRel
    FROM DealsWithProlong dwp
    JOIN DailyBaseRate   dbr
      ON dbr.DT_OPEN             = dwp.DT_OPEN
     AND dbr.SEG_NAME            = dwp.SEG_NAME
     AND dbr.PROD_NAME           = dwp.PROD_NAME
     AND dbr.CUR                 = dwp.CUR
     AND dbr.TermBucket          = dwp.TermBucket
     AND dbr.BalanceBucket       = dwp.BalanceBucket
     AND dbr.IS_OPTION           = dwp.IS_OPTION
     AND dbr.NEW_CONVENTION_NAME = dwp.NEW_CONVENTION_NAME
)
