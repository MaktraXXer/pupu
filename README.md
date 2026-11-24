SELECT
    client_group,
    COUNT_BIG(*) AS clients
FROM ALM_TEST.WORK.RK_Oct2026_ClientRegistry
GROUP BY client_group;

SELECT COUNT_BIG(*) AS total_clients
FROM ALM_TEST.WORK.RK_Oct2026_ClientRegistry;
