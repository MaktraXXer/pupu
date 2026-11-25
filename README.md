USE ALM_TEST;
SET NOCOUNT ON;

;WITH base AS (

    SELECT
        p.dt_rep,
        p.client_group,
        ISNULL(b.bank_group, p.bank_name_main) AS bank_name_main,
        p.bank_type,
        p.gr_size,
        p.payment_type,
        p.transaction_type,
        p.operation_type,
        p.payment_direction,

        /* Потенциальное удержание с проверкой версии */

        CASE
            WHEN f.sms_window = 1 AND f.push_window = 1
                THEN N'SMS + Push'
            WHEN f.sms_window = 1 AND f.push_unknown_window = 1
                THEN N'SMS + Push не определен'
            WHEN f.push_window = 1 AND f.sms_unknown_window = 1
                THEN N'Push + SMS не определена'
            WHEN f.sms_window = 1
                THEN N'SMS'
            WHEN f.push_window = 1
                THEN N'Push'
            WHEN f.sms_unknown_window = 1 AND f.push_unknown_window = 1
                THEN N'SMS и Push не определены'
            WHEN f.sms_unknown_window = 1
                THEN N'SMS не определена'
            WHEN f.push_unknown_window = 1
                THEN N'Push не определен'
            ELSE N'Нет'
        END AS eligible_group,

        /* Потенциальное удержание без проверки версии */

        CASE
            WHEN n.sms_window = 1 AND n.push_window = 1
                THEN N'SMS + Push'
            WHEN n.sms_window = 1
                THEN N'SMS'
            WHEN n.push_window = 1
                THEN N'Push'
            ELSE N'Нет'
        END AS eligible_without_version_control,

        /* Фактическое удержание */

        CASE
            WHEN f.sms_offer_window = 1 AND f.push_offer_window = 1
                THEN N'SMS + Push'
            WHEN f.sms_offer_window = 1
                THEN N'SMS'
            WHEN f.push_offer_window = 1
                THEN N'Push'
            ELSE N'Нет'
        END AS fact_group,

        p.amount_in,
        p.amount_out,
        p.count_in,
        p.count_out

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

    WHERE p.dt_rep BETWEEN '2026-10-01' AND '2026-11-04'

)

SELECT
    dt_rep,
    client_group,
    bank_name_main,
    bank_type,
    gr_size,
    payment_type,
    transaction_type,
    operation_type,
    payment_direction,

    eligible_group,
    eligible_without_version_control,
    fact_group,

    N'Все денежные потоки' AS raspoznano_type,

    CAST(SUM(amount_in) / 1000000000.0
        AS decimal(20,6)) AS incoming,

    CAST(SUM(amount_out) / 1000000000.0
        AS decimal(20,6)) AS outgoing,

    CAST(SUM(amount_in + amount_out) / 1000000000.0
        AS decimal(20,6)) AS saldo,

    SUM(count_in) AS count_in,
    SUM(count_out) AS count_out,
    SUM(count_in + count_out) AS count_total

FROM base

GROUP BY
    dt_rep,
    client_group,
    bank_name_main,
    bank_type,
    gr_size,
    payment_type,
    transaction_type,
    operation_type,
    payment_direction,
    eligible_group,
    eligible_without_version_control,
    fact_group

ORDER BY
    dt_rep,
    client_group,
    bank_name_main,
    gr_size

OPTION (MAXDOP 1, RECOMPILE);
