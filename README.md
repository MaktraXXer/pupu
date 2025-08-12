USE [ALM];
SET NOCOUNT ON;

DECLARE @Start    date = '2024-01-01';
DECLARE @End      date = '2025-07-31';  -- конец июля 2025
DECLARE @Special  date = '2025-08-10';  -- доп. дата

-- 0) Целевые даты: все month-end от @Start до @End + @Special
WITH months AS (
    SELECT 0 AS n, EOMONTH(@Start) AS dt_rep
    UNION ALL
    SELECT n + 1,
           EOMONTH(DATEADD(MONTH, n + 1, @Start))
    FROM months
    WHERE EOMONTH(DATEADD(MONTH, n + 1, @Start)) <= @End
),
target_dates AS (
    SELECT dt_rep FROM months
    UNION ALL
    SELECT @Special
),
src AS (  -- 1) Источник по датам и фильтрам
    SELECT
        t.dt_rep,
        t.cli_id,
        t.con_id,
        t.out_rub,
        t.TSEGMENTNAME
    FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
    JOIN target_dates d ON d.dt_rep = t.dt_rep
    WHERE t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND t.out_rub IS NOT NULL
      AND t.section_name IN (N'Накопительный счёт', N'Срочные ', N'До востребования')
      AND ISNULL(t.TSEGMENTNAME, N'0') IN (N'ДЧБО', N'Розничный бизнес', N'0')
),
-- 2) Агрегация по клиенту на каждую дату
client_totals AS (
    SELECT
        s.dt_rep,
        s.cli_id,
        SUM(CASE WHEN s.out_rub < 0 THEN 0 ELSE s.out_rub END) AS total_out_rub,
        MAX(CASE WHEN LTRIM(RTRIM(s.TSEGMENTNAME)) = N'ДЧБО' THEN 1 ELSE 0 END) AS has_dchbo
    FROM src s
    GROUP BY s.dt_rep, s.cli_id
),
-- 3) Сегмент и бакет
labeled AS (
    SELECT
        dt_rep,
        cli_id,
        total_out_rub,
        CASE WHEN has_dchbo = 1 THEN N'УЧК' ELSE N'Розница' END AS segment,
        CASE
            WHEN total_out_rub >=        0     AND total_out_rub <   1500000       THEN N'[0; 1.5 млн)'
            WHEN total_out_rub >=   1500000     AND total_out_rub <  15000000       THEN N'[1.5 млн; 15 млн)'
            WHEN total_out_rub >=  15000000     AND total_out_rub <= 3000000000000  THEN N'[15 млн; 3000 млрд]'
            ELSE N'вне диапазона'
        END AS bucket
    FROM client_totals
)
-- 4) Агрегат по датам
SELECT
    dt_rep,
    segment,
    bucket,
    COUNT(DISTINCT cli_id) AS clients_cnt,
    SUM(total_out_rub)     AS sum_out_rub
FROM labeled
WHERE bucket <> N'вне диапазона'
GROUP BY dt_rep, segment, bucket
ORDER BY dt_rep, segment, bucket;

OPTION (MAXRECURSION 200);
