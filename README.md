SELECT
    CASE
        WHEN has_app_activity = 0
            THEN N'Активность не найдена'
        WHEN has_unknown_app = 1
            THEN N'Версия или ОС не определена'
        ELSE N'Прочее'
    END AS reason,

    sms_unknown_day,
    push_unknown_day,

    SUM(outgoing_sr) / 1e9 AS outgoing_depo,
    SUM(outgoing_ns) / 1e9 AS outgoing_ns,
    SUM(outgoing_dvs) / 1e9 AS outgoing_dvs,

    SUM(outgoing_sr + outgoing_ns + outgoing_dvs)
        / 1e9 AS outgoing_total

FROM ALM_TEST.WORK.RK_Oct2026_PaymentAgg

WHERE client_group IN (
    N'старый клиент масс кто принес новые деньги',
    N'старый клиент масс кто не принес новые деньги'
)

AND sbp_self_1m = 1
AND sms_offer_window = 0
AND push_offer_window = 0

AND (
    sms_unknown_day = 1
    OR push_unknown_day = 1
)

GROUP BY
    CASE
        WHEN has_app_activity = 0
            THEN N'Активность не найдена'
        WHEN has_unknown_app = 1
            THEN N'Версия или ОС не определена'
        ELSE N'Прочее'
    END,
    sms_unknown_day,
    push_unknown_day

OPTION (MAXDOP 1);
