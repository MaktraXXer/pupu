USE ALM;

DECLARE @dt date = '2026-10-15';
DECLARE @LookbackDays int = 7;

;WITH clients AS (

    SELECT TOP (20)
        cli_id,
        push_unknown
    FROM #rk26w_event
    WHERE dt_rep = @dt
      AND sms_unknown = 1
      AND has_app_activity = 0
    ORDER BY cli_id

)

SELECT
    c.cli_id,
    c.push_unknown,

    MAX(CASE
        WHEN d.CLI_ID = c.cli_id
         AND d.LOG_TIMESTAMP >= @dt
        THEN 1 ELSE 0
    END) AS activity_same_day,

    MAX(CASE
        WHEN d.CLI_ID = c.cli_id
         AND d.LOG_TIMESTAMP < @dt
        THEN 1 ELSE 0
    END) AS activity_previous_days,

    MAX(CASE
        WHEN d.CLI_ID = -c.cli_id
        THEN 1 ELSE 0
    END) AS opposite_id_found

FROM clients c

LEFT JOIN ALM.ehd.ods_cli_devices d
    ON d.CLI_ID IN (c.cli_id, -c.cli_id)
   AND d.LOG_TIMESTAMP >= DATEADD(day, -@LookbackDays, @dt)
   AND d.LOG_TIMESTAMP < DATEADD(day, 1, @dt)

GROUP BY
    c.cli_id,
    c.push_unknown

OPTION (MAXDOP 1, RECOMPILE);
