USE [ALM];
SET NOCOUNT ON;

-- 1) Источник с фильтрами на дату и витрину
WITH src AS (
    SELECT
        t.cli_id,
        t.con_id,
        t.out_rub,
        t.TSEGMENTNAME
    FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
    WHERE t.dt_rep = '2025-08-10'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND t.out_rub IS NOT NULL
      AND t.section_name IN (N'Накопительный счёт', N'Срочные ', N'До востребования') -- обратите внимание на пробел в 'Срочные '
      AND ISNULL(t.TSEGMENTNAME, N'0') IN (N'ДЧБО', N'Розничный бизнес', N'0')
),
-- 2) Агрегация по клиенту: общий объём и флаг наличия ДЧБО
client_totals AS (
    SELECT
        s.cli_id,
        SUM(CASE WHEN s.out_rub < 0 THEN 0 ELSE s.out_rub END) AS total_out_rub,
        MAX(CASE WHEN s.TSEGMENTNAME = N'ДЧБО' THEN 1 ELSE 0 END) AS has_dchbo
    FROM src s
    GROUP BY s.cli_id
),
-- 3) Присвоение сегмента и бакета
labeled AS (
    SELECT
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
-- 4) Результат как временная таблица
IF OBJECT_ID('tempdb..#client_buckets', 'U') IS NOT NULL DROP TABLE #client_buckets;

SELECT
    segment,
    bucket,
    COUNT(DISTINCT cli_id)         AS clients_cnt,
    SUM(total_out_rub)             AS sum_out_rub
INTO #client_buckets
FROM labeled
WHERE bucket <> N'вне диапазона'
GROUP BY segment, bucket;

-- Посмотреть результат:
SELECT *
FROM #client_buckets
ORDER BY segment, bucket;
