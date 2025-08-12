USE [ALM];
SET NOCOUNT ON;

WITH src AS (
    SELECT
        t.dt_rep,
        t.cli_id,
        t.con_id,
        t.PROD_NAME_res,
        t.TSEGMENTNAME,
        CAST(t.out_rub AS decimal(20,2)) AS out_rub
    FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
    WHERE t.dt_rep = '2025-08-10'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND t.out_rub IS NOT NULL
      AND t.section_name IN (N'Накопительный счёт', N'Срочные ', N'До востребования') -- 'Срочные ' с пробелом
      AND ISNULL(t.TSEGMENTNAME, N'0') IN (N'ДЧБО', N'Розничный бизнес', N'0')
),
client_totals AS (       -- сумма по клиенту + флаг наличия ДЧБО
    SELECT
        s.cli_id,
        SUM(CASE WHEN s.out_rub < 0 THEN 0 ELSE s.out_rub END) AS total_out_rub,
        MAX(CASE WHEN LTRIM(RTRIM(s.TSEGMENTNAME)) = N'ДЧБО' THEN 1 ELSE 0 END) AS has_dchbo
    FROM src s
    GROUP BY s.cli_id
),
labeled AS (             -- сегмент клиента + бакет по сумме
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
),
uchk_min_bucket AS (     -- только УЧК с минимальным бакетом
    SELECT cli_id, total_out_rub, segment, bucket
    FROM labeled
    WHERE segment = N'УЧК' AND bucket = N'[0; 1.5 млн)'
)
SELECT
    u.segment,
    u.bucket,
    u.cli_id,
    u.total_out_rub                 AS total_out_rub_client,
    s.con_id,
    s.PROD_NAME_res,
    s.TSEGMENTNAME                  AS TSEGMENTNAME_raw,
    CAST(CASE WHEN s.out_rub < 0 THEN 0 ELSE s.out_rub END AS decimal(20,2)) AS out_rub_con
FROM uchk_min_bucket u
JOIN src s
  ON s.cli_id = u.cli_id
ORDER BY u.cli_id, out_rub_con DESC, s.con_id;
