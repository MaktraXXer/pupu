
IF NOT EXISTS (
    SELECT 1
    FROM #rk26w_job
    WHERE stage = 5
)
    THROW 50001, 'Расчет не завершен', 1;


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
    a.cli_biz,

    a.target_flag,
    a.sbp_self_1m,


    /* Фактическое окно на дату платежа */

    CASE
        WHEN a.sms_offer_window = 1
         AND a.push_offer_window = 1
            THEN N'SMS + Push'

        WHEN a.sms_offer_window = 1
            THEN N'SMS'

        WHEN a.push_offer_window = 1
            THEN N'Push'

        ELSE N'Нет'

    END AS fact_group_day,


    /* Потенциальное окно на дату платежа */

    CASE
        WHEN a.sms_window = 1
         AND a.push_window = 1
            THEN N'SMS + Push'

        WHEN a.sms_window = 1
            THEN N'SMS'

        WHEN a.push_window = 1
            THEN N'Push'

        WHEN a.sms_unknown_window = 1
          OR a.push_unknown_window = 1
            THEN N'Не определено'

        ELSE N'Нет'

    END AS eligible_group_day,


    /* Триггеры и приложения */

    a.trigger_day,

    a.has_app_activity,
    a.has_old_app,
    a.has_new_app,
    a.has_unknown_app,

    a.has_closed_deposit,


    /* Потенциальные права */

    a.sms_candidate_day,
    a.sms_repeat_blocked_day,

    a.sms_eligible_day,
    a.push_eligible_day,

    a.sms_unknown_day,
    a.push_unknown_day,


    /* Фактические уведомления */

    a.sms_fact_day,
    a.push_fact_day,

    a.push_fact_unmatched_day,


    /* Потенциальные окна */

    a.sms_window,
    a.push_window,

    a.sms_unknown_window,
    a.push_unknown_window,


    /* Фактические окна */

    a.sms_offer_window,
    a.push_offer_window,

    a.sms_cooldown_day,


    /* Денежные показатели */

    v.amount_scope,
    v.raspoznano_type,

    CASE
        WHEN v.amount_scope = N'Полные суммы'
            THEN a.source_rows
    END AS source_rows,

    CASE
        WHEN v.amount_scope = N'Полные суммы'
            THEN a.count_in
    END AS count_in,

    CASE
        WHEN v.amount_scope = N'Полные суммы'
            THEN a.count_out
    END AS count_out,

    CAST(
        v.incoming / 1000000000.0
        AS decimal(20,6)
    ) AS incoming,

    CAST(
        v.outgoing / 1000000000.0
        AS decimal(20,6)
    ) AS outgoing,

    CAST(
        (v.incoming + v.outgoing) / 1000000000.0
        AS decimal(20,6)
    ) AS saldo

FROM #rk26w_agg a

CROSS APPLY (

    VALUES

        (
            N'Компоненты',
            N'Вклады',
            a.incoming_sr,
            a.outgoing_sr
        ),

        (
            N'Компоненты',
            N'НС',
            a.incoming_ns,
            a.outgoing_ns
        ),

        (
            N'Компоненты',
            N'ДВС',
            a.incoming_dvs,
            a.outgoing_dvs
        ),

        (
            N'Компоненты',
            N'Другие распознанные',
            a.incoming_other,
            a.outgoing_other
        ),

        (
            N'Компоненты',
            N'Нераспознанные',
            a.incoming_unknown,
            a.outgoing_unknown
        ),

        (
            N'Полные суммы',
            N'Полные переводы',
            a.amount_in,
            a.amount_out
        )

) v(amount_scope, raspoznano_type, incoming, outgoing)

WHERE v.incoming <> 0
   OR v.outgoing <> 0
   OR (
       v.amount_scope = N'Полные суммы'
       AND a.source_rows > 0
   )

ORDER BY
    a.dt_rep,
    a.client_group,
    a.bank_name_main,
    a.gr_size,
    v.amount_scope,
    v.raspoznano_type

OPTION (MAXDOP 1, RECOMPILE);
