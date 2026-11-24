USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @DateFrom date = '2026-10-01';
DECLARE @DateTo date = '2026-11-04';
DECLARE @dt date = DATEADD(day, -6, @DateFrom);

IF OBJECT_ID('tempdb..#rk26w_event') IS NULL
   OR OBJECT_ID('tempdb..#rk26p_cli') IS NULL
   OR OBJECT_ID('tempdb..#rk26w_dayflags') IS NULL
   OR OBJECT_ID('tempdb..#rk26w_bank') IS NULL
    THROW 50001, 'Не найдены временные таблицы', 1;

IF NOT EXISTS (
    SELECT 1 FROM #rk26w_job WHERE stage = 5
)
    THROW 50002, 'Расчет не завершен', 1;

DROP TABLE IF EXISTS #rk26nv_sms;
DROP TABLE IF EXISTS #rk26nv_day;


/* Потенциальные SMS без проверки версии приложения */

CREATE TABLE #rk26nv_sms (
    cli_id bigint NOT NULL,
    dt_rep date NOT NULL,
    PRIMARY KEY (cli_id, dt_rep)
);

WHILE @dt <= @DateTo
BEGIN

    INSERT INTO #rk26nv_sms

    SELECT
        t.cli_id,
        @dt

    FROM #rk26w_event t

    WHERE t.dt_rep = @dt
      AND t.trigger_rows > 0
      AND t.sms_cooldown = 0

      AND NOT EXISTS (
          SELECT 1
          FROM #rk26nv_sms s
          WHERE s.cli_id = t.cli_id
            AND s.dt_rep BETWEEN DATEADD(day, -6, @dt)
                             AND DATEADD(day, -1, @dt)
      )

    OPTION (MAXDOP 1, RECOMPILE);

    SET @dt = DATEADD(day, 1, @dt);

END;


/* Потенциальные окна без проверки приложения */

;WITH nums AS (

    SELECT n
    FROM (VALUES (0),(1),(2),(3),(4),(5),(6)) v(n)

),

windows AS (

    /* SMS: первый триггер + 6 дней */

    SELECT
        s.cli_id,
        DATEADD(day, n.n, s.dt_rep) AS dt_rep,
        1 AS sms_window,
        0 AS push_window

    FROM #rk26nv_sms s
    CROSS JOIN nums n

    UNION ALL

    /* Push: от триггера до закрытия + 2 дня */

    SELECT
        t.cli_id,
        DATEADD(day, n.n, t.dt_rep),
        0,
        1

    FROM #rk26w_event t
    CROSS JOIN nums n

    WHERE t.trigger_rows > 0
      AND t.close_dt IS NOT NULL
      AND n.n <= 2

      AND DATEADD(day, n.n, t.dt_rep)
          <= DATEADD(day, 2, t.close_dt)

)

SELECT
    cli_id,
    dt_rep,
    MAX(sms_window) AS sms_window,
    MAX(push_window) AS push_window

INTO #rk26nv_day

FROM windows

WHERE dt_rep BETWEEN @DateFrom AND @DateTo

GROUP BY cli_id, dt_rep

OPTION (MAXDOP 1, RECOMPILE);


CREATE UNIQUE CLUSTERED INDEX IX_rk26nv_day
ON #rk26nv_day (dt_rep, cli_id);


/* Маркировка платежей */

;WITH base AS (

    SELECT
        p.*,

        ISNULL(b.bank_group, p.bank_name_main)
            AS bank_group,

        /* Потенциальное удержание с учетом версии */

        CASE
            WHEN f.sms_window = 1
             AND f.push_window = 1
                THEN N'SMS + Push'

            WHEN f.sms_window = 1
             AND f.push_unknown_window = 1
                THEN N'SMS + Push не определен'

            WHEN f.push_window = 1
             AND f.sms_unknown_window = 1
                THEN N'Push + SMS не определена'

            WHEN f.sms_window = 1
                THEN N'SMS'

            WHEN f.push_window = 1
                THEN N'Push'

            WHEN f.sms_unknown_window = 1
             AND f.push_unknown_window = 1
                THEN N'SMS и Push не определены'

            WHEN f.sms_unknown_window = 1
                THEN N'SMS не определена'

            WHEN f.push_unknown_window = 1
                THEN N'Push не определен'

            ELSE N'Нет'

        END AS eligible_group,

        /* Потенциальное удержание без версии */

        CASE
            WHEN n.sms_window = 1
             AND n.push_window = 1
                THEN N'SMS + Push'

            WHEN n.sms_window = 1
                THEN N'SMS'

            WHEN n.push_window = 1
                THEN N'Push'

            ELSE N'Нет'

        END AS eligible_without_version_control,

        /* Фактическое удержание */

        CASE
            WHEN f.sms_offer_window = 1
             AND f.push_offer_window = 1
                THEN N'SMS + Push'

            WHEN f.sms_offer_window = 1
                THEN N'SMS'

            WHEN f.push_offer_window = 1
                THEN N'Push'

            ELSE N'Нет'

        END AS fact_group

    FROM #rk26p_cli p

    LEFT JOIN #rk26w_dayflags f
        ON f.cli_id = p.cli_id
       AND f.dt_rep = p.dt_rep

    LEFT JOIN #rk26nv_day n
        ON n.cli_id = p.cli_id
       AND n.dt_rep = p.dt_rep

    LEFT JOIN #rk26w_bank b
        ON b.bank_name_main = p.bank_name_main
       AND p.bank_type = 'bank'

    WHERE p.dt_rep BETWEEN @DateFrom AND @DateTo

),

/* Агрегация */

agg AS (

    SELECT
        dt_rep,
        client_group,
        bank_group AS bank_name_main,
        bank_type,
        gr_size,
        payment_type,
        transaction_type,
        operation_type,
        payment_direction,

        eligible_group,
        eligible_without_version_control,
        fact_group,

        SUM(incoming_sr) AS incoming_sr,
        SUM(outgoing_sr) AS outgoing_sr,

        SUM(incoming_ns) AS incoming_ns,
        SUM(outgoing_ns) AS outgoing_ns,

        SUM(incoming_dvs) AS incoming_dvs,
        SUM(outgoing_dvs) AS outgoing_dvs,

        SUM(incoming_other) AS incoming_other,
        SUM(outgoing_other) AS outgoing_other,

        SUM(incoming_unknown) AS incoming_unknown,
        SUM(outgoing_unknown) AS outgoing_unknown,

        SUM(amount_in) AS amount_in,
        SUM(amount_out) AS amount_out

    FROM base

    GROUP BY
        dt_rep,
        client_group,
        bank_group,
        bank_type,
        gr_size,
        payment_type,
        transaction_type,
        operation_type,
        payment_direction,
        eligible_group,
        eligible_without_version_control,
        fact_group

)

/* Итог */

SELECT
    a.dt_rep,
    a.client_group,
    a.bank_name_main,
    a.bank_type,
    a.gr_size,
    a.payment_type,
    a.transaction_type,
    a.operation_type,
    a.payment_direction,

    a.eligible_group,
    a.eligible_without_version_control,
    a.fact_group,

    v.raspoznano_type,

    CAST(v.incoming / 1000000000.0
        AS decimal(20,6)) AS incoming,

    CAST(v.outgoing / 1000000000.0
        AS decimal(20,6)) AS outgoing,

    CAST((v.incoming + v.outgoing) / 1000000000.0
        AS decimal(20,6)) AS saldo

FROM agg a

CROSS APPLY (
    VALUES
        (N'Вклады', a.incoming_sr, a.outgoing_sr),
        (N'НС', a.incoming_ns, a.outgoing_ns),
        (N'ДВС', a.incoming_dvs, a.outgoing_dvs),
        (N'Другие распознанные', a.incoming_other, a.outgoing_other),
        (N'Нераспознанные', a.incoming_unknown, a.outgoing_unknown),
        (N'Полные переводы', a.amount_in, a.amount_out)

) v(raspoznano_type, incoming, outgoing)

WHERE v.incoming <> 0
   OR v.outgoing <> 0

ORDER BY
    a.dt_rep,
    a.client_group,
    a.bank_name_main,
    a.gr_size,
    v.raspoznano_type

OPTION (MAXDOP 1, RECOMPILE);
