USE [ALM];
SET NOCOUNT ON;

DECLARE @Start   date = '2024-01-01';
DECLARE @End     date = '2025-07-31';   -- конец июля 2025
DECLARE @Special date = '2025-08-10';   -- доп. дата

/* 1) Целевые даты: все month-end + спец-дата (без рекурсии) */
IF OBJECT_ID('tempdb..#dates','U') IS NOT NULL DROP TABLE #dates;
WITH tally AS (
    SELECT TOP (1 + DATEDIFF(MONTH, @Start, @End))
           ROW_NUMBER() OVER (ORDER BY (SELECT 1)) - 1 AS n
    FROM sys.all_objects
)
SELECT dt_rep
INTO #dates
FROM (
    SELECT EOMONTH(DATEADD(MONTH, n, @Start)) AS dt_rep FROM tally
    UNION ALL
    SELECT @Special
) d;
CREATE UNIQUE CLUSTERED INDEX IX_dates ON #dates(dt_rep);

/* 2) Агрегация клиента на дату (внутри каждой даты) */
IF OBJECT_ID('tempdb..#client_on_date','U') IS NOT NULL DROP TABLE #client_on_date;
SELECT
    t.dt_rep,
    t.cli_id,
    SUM(CASE WHEN t.out_rub < 0 THEN 0 ELSE t.out_rub END) AS total_out_rub,
    MAX(CASE WHEN LTRIM(RTRIM(t.TSEGMENTNAME)) = N'ДЧБО' THEN 1 ELSE 0 END) AS has_dchbo
INTO #client_on_date
FROM alm.ALM.vw_balance_rest_all AS t WITH (NOLOCK)
JOIN #dates d ON d.dt_rep = t.dt_rep
WHERE
    t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag  = 1
    AND t.cur      = '810'
    AND t.out_rub IS NOT NULL
    AND t.section_name IN (N'Накопительный счёт', N'Срочные', N'Срочные ', N'До востребования')
    AND (t.TSEGMENTNAME IN (N'ДЧБО', N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
GROUP BY t.dt_rep, t.cli_id;

CREATE CLUSTERED INDEX IX_client_on_date ON #client_on_date(dt_rep, cli_id);

/* 3) Финальный агрегат по датам/сегментам/бакетам */
SELECT
    c.dt_rep,
    v.segment,
    v.bucket,
    COUNT(*)                 AS clients_cnt,   -- по одной строке на клиента → COUNT(*) = число клиентов
    SUM(c.total_out_rub)     AS sum_out_rub
FROM #client_on_date AS c
CROSS APPLY (
    VALUES (
        CASE WHEN c.has_dchbo = 1 THEN N'УЧК' ELSE N'Розница' END,
        CASE
            WHEN c.total_out_rub >=         0    AND c.total_out_rub <   1000000        THEN N'[0; 1 млн)'
            WHEN c.total_out_rub >=   1000000    AND c.total_out_rub <   1500000        THEN N'[1; 1.5 млн)'
            WHEN c.total_out_rub >=   1500000    AND c.total_out_rub <  15000000        THEN N'[1.5 млн; 15 млн)'
            WHEN c.total_out_rub >=  15000000    AND c.total_out_rub <= 3000000000000   THEN N'[15 млн; 3000 млрд]'
            ELSE N'вне диапазона'
        END
    )
) v(segment, bucket)
WHERE
    c.total_out_rub >= 0
    AND v.bucket <> N'вне диапазона'
GROUP BY
    c.dt_rep, v.segment, v.bucket
ORDER BY
    c.dt_rep, v.segment, v.bucket;
