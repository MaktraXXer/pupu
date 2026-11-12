
;WITH client_summary AS (
    SELECT
        c.CLI_ID,
        ISNULL(SUM(t.out_rub), 0) AS exit_td_sum,
        COUNT(t.CON_ID) AS exit_td_count
    FROM #clients c
    LEFT JOIN #tagged t
        ON t.CLI_ID = c.CLI_ID
       AND t.deal_type = 'EXIT'
    GROUP BY c.CLI_ID
)
SELECT
    (SELECT SUM(out_rub)
     FROM #tagged
     WHERE deal_type = 'EXIT') AS [Посделочная сумма],

    (SELECT SUM(exit_td_sum)
     FROM client_summary) AS [Поклиентная сумма],

    (SELECT COUNT(*)
     FROM #tagged
     WHERE deal_type = 'EXIT') AS [Посделочно строк],

    (SELECT SUM(exit_td_count)
     FROM client_summary) AS [Поклиентно договоров],

    (SELECT SUM(out_rub)
     FROM #snap
     WHERE dt_rep = '2026-10-19'
       AND section_name = N'Срочные'
       AND dt_close BETWEEN '2026-10-20' AND '2026-11-10'
       AND out_rub > 0) AS [Исходный баланс к выходу];
