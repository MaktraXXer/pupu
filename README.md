;WITH clients AS (
    SELECT *
    FROM ALM_TEST.alm_report.depo_fl_client_monthly_stats
    WHERE segment_flag <> N'ДЧБО'
      AND client_base_type = N'02. НС без вкладов к выходу'
      AND observation_month = '2026-10-01'
      AND is_sms_sent = 0
),

flags AS (
    SELECT
        b.cli_id,

        1 AS has_old_app_ever,

        MAX(CASE
            WHEN b.LOG_TIMESTAMP >= '2026-10-01'
             AND b.LOG_TIMESTAMP <  '2026-11-01'
                THEN 1 ELSE 0
        END) AS has_old_app_october

    FROM ALM.ehd.ods_cli_devices b WITH (NOLOCK)

    JOIN clients a
        ON a.cli_id = b.cli_id

    CROSS APPLY (
        SELECT LTRIM(RTRIM(b.APP_VERSION)) AS app
    ) s

    CROSS APPLY (
        SELECT CASE
            WHEN LEN(s.app) - LEN(REPLACE(s.app,'.','')) = 2
                THEN s.app + '.0'
            WHEN LEN(s.app) - LEN(REPLACE(s.app,'.','')) = 3
                THEN s.app
        END AS ver
    ) v

    CROSS APPLY (
        SELECT
            TRY_CAST(PARSENAME(v.ver,4) AS int) AS major,
            TRY_CAST(PARSENAME(v.ver,3) AS int) AS minor,
            TRY_CAST(PARSENAME(v.ver,2) AS int) AS patch,
            TRY_CAST(PARSENAME(v.ver,1) AS int) AS build
    ) p

    WHERE p.major >= 0
      AND p.minor >= 0
      AND p.patch >= 0
      AND p.build >= 0

      AND (
          p.major < 5
          OR (p.major = 5 AND p.minor < 7)
          OR (p.major = 5 AND p.minor = 7 AND p.patch < 1)
      )

    GROUP BY b.cli_id
)

SELECT
    a.*,
    ISNULL(f.has_old_app_ever, 0) AS has_old_app_ever,
    ISNULL(f.has_old_app_october, 0) AS has_old_app_october

FROM clients a

LEFT JOIN flags f
    ON a.cli_id = f.cli_id

OPTION (MAXDOP 1, RECOMPILE);
