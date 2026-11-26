SELECT a.*, b.*

FROM ALM_TEST.alm_report.depo_fl_client_monthly_stats a

OUTER APPLY (
    SELECT TOP (1) d.*
    FROM ALM.ehd.ods_cli_devices d WITH (NOLOCK)
    WHERE d.cli_id = a.cli_id
      AND d.LOG_TIMESTAMP < DATEADD(day, 1, a.analysis_date)
    ORDER BY d.LOG_TIMESTAMP DESC
) b

CROSS APPLY (
    SELECT CASE
        WHEN LEN(b.APP_VERSION)
           - LEN(REPLACE(b.APP_VERSION, '.', '')) = 2
            THEN b.APP_VERSION + '.0'
        ELSE b.APP_VERSION
    END AS ver
) v

CROSS APPLY (
    SELECT
        TRY_CONVERT(int, PARSENAME(v.ver, 4)) AS major,
        TRY_CONVERT(int, PARSENAME(v.ver, 3)) AS minor,
        TRY_CONVERT(int, PARSENAME(v.ver, 2)) AS patch,
        TRY_CONVERT(int, PARSENAME(v.ver, 1)) AS build
) p

WHERE a.segment_flag <> N'ДЧБО'
  AND a.client_base_type = N'02. НС без вкладов к выходу'
  AND a.observation_month = '2026-10-01'
  AND a.is_sms_sent = 0

  AND b.cli_id IS NOT NULL

  AND (
      b.OS IS NULL
      OR (
          LOWER(LTRIM(RTRIM(b.OS))) IN ('android', 'ios')
          AND p.major >= 0
          AND p.minor >= 0
          AND p.patch >= 0
          AND p.build >= 0
          AND (
              p.major > 5
              OR (p.major = 5 AND p.minor > 7)
              OR (p.major = 5 AND p.minor = 7 AND p.patch >= 1)
          )
      )
  )

OPTION (MAXDOP 1, RECOMPILE);
