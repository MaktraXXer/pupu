IF OBJECT_ID('tempdb..#z0') IS NOT NULL DROP TABLE #z0;

;WITH z_ranked AS (
    SELECT
        z.*,
        ROW_NUMBER() OVER (
            PARTITION BY z.con_id
            ORDER BY z.rate_con DESC
        ) AS rn
    FROM ALM.ehd.import_ZeroContracts z WITH (NOLOCK)
    WHERE z.dt_open <= @Anchor
      AND NOT EXISTS (
          SELECT 1
          FROM #bal_prom p
          WHERE p.con_id = z.con_id
      )
)
SELECT
    CAST(z.con_id AS bigint) AS con_id,
    CAST(z.cli_id AS bigint) AS cli_id,
    CAST(0.00 AS decimal(20,2)) AS out_rub,

    CAST(
        COALESCE(
            z.rate_con,
            CASE
                WHEN LTRIM(RTRIM(COALESCE(z.TSEGMENTNAME,N''))) = N'ДЧБО'
                    THEN @PromoRate_DChbo
                ELSE @PromoRate_Retail
            END
        ) AS decimal(9,4)
    ) AS rate_anchor,

    CAST(z.dt_open AS date) AS dt_open,

    CASE
        WHEN LTRIM(RTRIM(COALESCE(z.TSEGMENTNAME,N''))) = N''
            THEN N'Розничный бизнес'
        ELSE z.TSEGMENTNAME
    END AS TSEGMENTNAME

INTO #z0
FROM z_ranked z
WHERE z.rn = 1;
