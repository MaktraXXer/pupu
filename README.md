
SELECT
    p.period_name,

    COUNT(DISTINCT CASE
        WHEN n.CLI_ID IS NOT NULL
        THEN s.CLI_ID END
    ) AS new_clients_in_second,

    COUNT(DISTINCT CASE
        WHEN m.CLI_ID IS NOT NULL
        THEN s.CLI_ID END
    ) AS overlapping_clients

FROM #nm_selected s

JOIN #periods p
    ON p.period_id = s.period_id

LEFT JOIN #new_clients n
    ON n.period_id = s.period_id
   AND n.CLI_ID = s.CLI_ID

LEFT JOIN #mass_clients m
    ON m.period_id = s.period_id
   AND m.CLI_ID = s.CLI_ID

GROUP BY p.period_id, p.period_name
ORDER BY p.period_id;



;WITH selected AS (
    SELECT
        N'Новые клиенты' AS portfolio,
        period_id,
        CON_ID
    FROM #contracts

    UNION ALL

    SELECT
        N'Остальные клиенты',
        period_id,
        CON_ID
    FROM #nm_selected
)
SELECT
    p.period_name,
    s.portfolio,
    o.PROD_NAME,
    COUNT(DISTINCT s.CON_ID) AS contracts
FROM selected s
JOIN #opened o
    ON o.period_id = s.period_id
   AND o.CON_ID = s.CON_ID
JOIN #periods p
    ON p.period_id = s.period_id
WHERE o.PROD_NAME IN (
    N'Всё в ДОМ',
    N'НадёжныйVIP',
    N'Надёжныйпроцент',
    N'Надёжныйпромо',
    N'Надёжныйпрайм'
)
GROUP BY
    p.period_id,
    p.period_name,
    s.portfolio,
    o.PROD_NAME
ORDER BY p.period_id, s.portfolio;
