totals AS
(
    SELECT
          bucket_code
        , segment_name
        , is_total
        , segment_vol

        /* Общий знаменатель:
           отдельно private, отдельно все остальные категории */
        , SUM(
            CASE
                WHEN is_total = 0 THEN segment_vol
                ELSE 0
            END
          ) OVER
          (
              PARTITION BY
                  CASE
                      WHEN segment_name = N'вклад private-сегмента'
                          THEN 1
                      ELSE 0
                  END
          ) AS segment_total

        , MAX(
            CASE
                WHEN is_total = 1 THEN segment_vol
            END
          ) OVER
          (
              PARTITION BY bucket_code
          ) AS bucket_total

        , SUM(
            CASE
                WHEN is_total = 1 THEN segment_vol
                ELSE 0
            END
          ) OVER () AS grand_total

    FROM agg
)
