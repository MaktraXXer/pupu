;WITH base AS (

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

        /* Потенциальное удержание */

        CASE
            WHEN a.sms_window = 1
             AND a.push_window = 1
                THEN N'SMS + Push'

            WHEN a.sms_window = 1
             AND a.push_unknown_window = 1
                THEN N'SMS + Push не определен'

            WHEN a.push_window = 1
             AND a.sms_unknown_window = 1
                THEN N'Push + SMS не определена'

            WHEN a.sms_window = 1
                THEN N'SMS'

            WHEN a.push_window = 1
                THEN N'Push'

            WHEN a.sms_unknown_window = 1
             AND a.push_unknown_window = 1
                THEN N'SMS и Push не определены'

            WHEN a.sms_unknown_window = 1
                THEN N'SMS не определена'

            WHEN a.push_unknown_window = 1
                THEN N'Push не определен'

            ELSE N'Нет'

        END AS eligible_group,

        /* Фактическое удержание */

        CASE
            WHEN a.sms_offer_window = 1
             AND a.push_offer_window = 1
                THEN N'SMS + Push'

            WHEN a.sms_offer_window = 1
                THEN N'SMS'

            WHEN a.push_offer_window = 1
                THEN N'Push'

            ELSE N'Нет'

        END AS fact_group,

        a.incoming_sr,
        a.outgoing_sr,

        a.incoming_ns,
        a.outgoing_ns,

        a.incoming_dvs,
        a.outgoing_dvs,

        a.incoming_other,
        a.outgoing_other,

        a.incoming_unknown,
        a.outgoing_unknown,

        a.amount_in,
        a.amount_out

    FROM ALM_TEST.WORK.RK_Oct2026_PaymentAgg a

),

/* Агрегация */

agg AS (

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
        bank_name_main,
        bank_type,
        gr_size,
        payment_type,
        transaction_type,
        operation_type,
        payment_direction,
        eligible_group,
        fact_group

)

/* Итоговая выгрузка */

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
