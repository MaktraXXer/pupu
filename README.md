SELECT
    a.*,

    CASE
        WHEN ISNULL(f.has_version_ever, 0) = 0 THEN 1
        ELSE f.has_old_app_ever
    END AS has_old_app_ever,

    CASE
        WHEN ISNULL(f.has_version_october, 0) = 0 THEN 1
        ELSE f.has_old_app_october
    END AS has_old_app_october

FROM ALM_TEST.alm_report.depo_fl_client_monthly_stats a

LEFT JOIN (

    SELECT
        b.CLI_ID AS device_cli_id,

        MAX(CASE WHEN x.app_class >= 0
                 THEN 1 ELSE 0 END) AS has_version_ever,

        MAX(CASE WHEN x.app_class = 1
                 THEN 1 ELSE 0 END) AS has_old_app_ever,

        MAX(CASE WHEN x.app_class >= 0
                  AND b.LOG_TIMESTAMP >= '20261001'
                  AND b.LOG_TIMESTAMP < '20261101'
                 THEN 1 ELSE 0 END) AS has_version_october,

        MAX(CASE WHEN x.app_class = 1
                  AND b.LOG_TIMESTAMP >= '20261001'
                  AND b.LOG_TIMESTAMP < '20261101'
                 THEN 1 ELSE 0 END) AS has_old_app_october

    FROM ALM.ehd.ods_cli_devices b WITH (NOLOCK)

    JOIN ALM_TEST.alm_report.depo_fl_client_monthly_stats c
        ON c.cli_id = b.CLI_ID
       AND c.observation_month = '20261001'
       AND c.segment_flag <> N'ДЧБО'
       AND c.client_base_type = N'02. НС без вкладов к выходу'
       AND c.is_sms_sent = 0

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

    CROSS APPLY (
        SELECT CASE
            WHEN p.major IS NULL
              OR p.minor IS NULL
              OR p.patch IS NULL
              OR p.build IS NULL
              OR p.major < 0
              OR p.minor < 0
              OR p.patch < 0
              OR p.build < 0
                THEN -1

            WHEN p.major < 5
              OR (p.major = 5 AND p.minor < 7)
              OR (p.major = 5 AND p.minor = 7 AND p.patch < 1)
                THEN 1

            ELSE 0
        END AS app_class
    ) x

    GROUP BY b.CLI_ID

) f ON a.cli_id = f.device_cli_id

WHERE a.segment_flag <> N'ДЧБО'
  AND a.client_base_type = N'02. НС без вкладов к выходу'
  AND a.observation_month = '20261001'
  AND a.is_sms_sent = 0

OPTION (MAXDOP 1, RECOMPILE);
