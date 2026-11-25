USE ALM_TEST;
SET NOCOUNT ON;

;WITH base AS (

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

        CASE
            WHEN sms_window = 1 AND push_window = 1
                THEN N'SMS + Push'
            WHEN sms_window = 1 AND push_unknown_window = 1
                THEN N'SMS + Push не определен'
            WHEN push_window = 1 AND sms_unknown_window = 1
                THEN N'Push + SMS не определена'
            WHEN sms_window = 1
                THEN N'SMS'
            WHEN push_window = 1
                THEN N'Push'
            WHEN sms_unknown_window = 1 AND push_unknown_window = 1
                THEN N'SMS и Push не определены'
            WHEN sms_unknown_window = 1
                THEN N'SMS не определена'
            WHEN push_unknown_window = 1
                THEN N'Push не определен'
            ELSE N'Нет'
        END AS eligible_group,

        CASE
            WHEN sms_offer_window = 1 AND push_offer_window = 1
                THEN N'SMS + Push'
            WHEN sms_offer_window = 1
                THEN N'SMS'
            WHEN push_offer_window = 1
                THEN N'Push'
            ELSE N'Нет'
        END AS fact_group,

        amount_in,
        amount_out,
        count_in,
        count_out

    FROM WORK.RK_Oct2026_PaymentAgg

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
    SUM(count_in) + SUM(count_out) AS count_total

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

ORDER BY
    dt_rep,
    client_group,
    bank_name_main,
    gr_size

OPTION (MAXDOP 1, RECOMPILE);
