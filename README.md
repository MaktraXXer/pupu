SELECT dk.*,
       CA.BaseRate
INTO   #DailyBaseRate        -- временная таблица
FROM   DealKeys dk
CROSS APPLY (
    SELECT TOP (1) bc.BaseRate
    FROM   BaseCandidates bc
    JOIN   BalanceBuckets bbSrc
           ON bc.BalanceBucket = bbSrc.BALANCE_GROUP
    WHERE  bc.SEG_NAME             = dk.SEG_NAME
      AND  bc.PROD_NAME            = dk.PROD_NAME
      AND  bc.CUR                  = dk.CUR
      AND  bc.TermBucket           = dk.TermBucket
      AND  bc.IS_OPTION            = dk.IS_OPTION
      AND  bc.NEW_CONVENTION_NAME  = dk.NEW_CONVENTION_NAME
      /* окно по датам ±@DayWindow */
      AND  ABS(DATEDIFF(DAY, bc.DT_OPEN, dk.DT_OPEN)) <= @DayWindow
      /* направление по бакетам */
      AND (
            (dk.BucketUpper <= 1500000  AND bbSrc.BucketOrder <= dk.BucketOrder)  -- вниз
         OR (dk.BucketUpper  > 1500000  AND bbSrc.BucketOrder >= dk.BucketOrder)  -- вверх
          )
    ORDER BY
         ABS(DATEDIFF(DAY, bc.DT_OPEN, dk.DT_OPEN)) ASC,           -- ближе по дате
         CASE WHEN dk.BucketUpper <= 1500000                       -- ближе по объёму
              THEN dk.BucketOrder - bbSrc.BucketOrder              -- вниз
              ELSE bbSrc.BucketOrder - dk.BucketOrder              -- вверх
         END ASC
) CA;
