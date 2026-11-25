SELECT TOP (30)
    cli_id,
    action_date,
    should_be,
    total_out / 1e6 AS original_out_mln,

    COUNT_BIG(*) AS rows_count,
    COUNT(DISTINCT ping_date) AS message_dates,

    SUM(total_out) / 1e6 AS summed_out_mln,

    (SUM(total_out) - MAX(total_out)) / 1e6
        AS duplicated_out_mln

FROM ALM_TEST.alm_history.sms_push_clients

GROUP BY
    cli_id,
    action_date,
    should_be,
    total_out

HAVING COUNT_BIG(*) > 1

ORDER BY ABS(SUM(total_out) - MAX(total_out)) DESC

OPTION (MAXDOP 1);


Посмотри, пожалуйста, нет ли задвоения total_out при присоединении уведомлений и all_info. В последнем случае, если у клиента были и SMS, и Push, при JOIN по CLI_ID строки могут размножаться. Написал небольшой проверочный скрипт, который покажет конкретные примеры таких клиентов.
