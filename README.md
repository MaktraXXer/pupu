USE ALM_TEST;
SET NOCOUNT ON;

IF OBJECT_ID('tempdb..#rk26w_agg') IS NULL
    THROW 50001, 'Не найдены готовые агрегаты', 1;

IF NOT EXISTS (
    SELECT 1
    FROM #rk26w_job
    WHERE stage = 5
)
    THROW 50002, 'Расчет не завершен', 1;

IF OBJECT_ID('WORK.RK_Oct2026_PaymentAgg') IS NOT NULL
    THROW 50003, 'Таблица уже существует', 1;


/* Сохраняем все агрегаты и флаги */

SELECT *
INTO WORK.RK_Oct2026_PaymentAgg
FROM #rk26w_agg
OPTION (MAXDOP 1);


/* Проверка сохранения */

SELECT
    COUNT_BIG(*) AS rows_count,
    MIN(dt_rep) AS date_from,
    MAX(dt_rep) AS date_to,
    SUM(amount_in) / 1e9 AS incoming,
    SUM(amount_out) / 1e9 AS outgoing
FROM WORK.RK_Oct2026_PaymentAgg
OPTION (MAXDOP 1);
