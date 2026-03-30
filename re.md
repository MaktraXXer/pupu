Вот обе версии с добавленным segment_name по правилу:
	•	если у клиента хотя бы один счет в скоупе имел TSEGMENTNAME = N'ДЧБО', то сегмент клиента = ДЧБО
	•	иначе = Розничный бизнес

Я беру сегмент из #bal по клиенту в рамках имеющихся двух срезов.

⸻

1. НС на начало и конец по клиентам с сегментом

DECLARE @MondayStart date = '2026-03-23';
DECLARE @FactEndDate date = '2026-03-27';

;WITH client_segment AS
(
    SELECT
          b.cli_id
        , CASE
            WHEN MAX(CASE WHEN b.TSEGMENTNAME = N'ДЧБО' THEN 1 ELSE 0 END) = 1
                THEN N'ДЧБО'
            ELSE N'Розничный бизнес'
          END AS segment_name
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    GROUP BY b.cli_id
)
SELECT
      b.cli_id
    , s.segment_name
    , SUM(CASE WHEN b.dt_rep = @MondayStart THEN b.out_rub ELSE 0 END) AS ns_start_rub
    , SUM(CASE WHEN b.dt_rep = @FactEndDate THEN b.out_rub ELSE 0 END) AS ns_end_rub
    , SUM(CASE WHEN b.dt_rep = @FactEndDate THEN b.out_rub ELSE 0 END)
      - SUM(CASE WHEN b.dt_rep = @MondayStart THEN b.out_rub ELSE 0 END) AS ns_delta_rub
FROM #bal b
INNER JOIN #clients_scope c
    ON b.cli_id = c.cli_id
LEFT JOIN client_segment s
    ON b.cli_id = s.cli_id
WHERE b.section_name = N'Накопительный счёт'
GROUP BY
      b.cli_id
    , s.segment_name
ORDER BY b.cli_id;


⸻

2. Сводная деталка по клиенту с сегментом

DECLARE @MondayStart date = '2026-03-23';
DECLARE @FactEndDate date = '2026-03-27';

DECLARE @WeekFrom date = DATEADD(day, 1, @MondayStart);
DECLARE @WeekTo   date = @FactEndDate;

;WITH client_segment AS
(
    SELECT
          b.cli_id
        , CASE
            WHEN MAX(CASE WHEN b.TSEGMENTNAME = N'ДЧБО' THEN 1 ELSE 0 END) = 1
                THEN N'ДЧБО'
            ELSE N'Розничный бизнес'
          END AS segment_name
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    GROUP BY b.cli_id
),
exit_agg AS
(
    SELECT
          b.cli_id
        , COUNT(DISTINCT b.con_id) AS cnt_con_exit
        , SUM(b.out_rub) AS vol_exit_rub
        , CAST(
            SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END)
            / NULLIF(SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_exit
    FROM #bal b
    WHERE b.dt_rep = @MondayStart
      AND b.section_name = N'Срочные'
      AND b.dt_close_plan >= @WeekFrom
      AND b.dt_close_plan <= @WeekTo
    GROUP BY b.cli_id
),
open_agg AS
(
    SELECT
          b.cli_id
        , COUNT(DISTINCT b.con_id) AS cnt_con_open
        , SUM(b.out_rub) AS vol_open_rub
        , CAST(
            SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END)
            / NULLIF(SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_open
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.dt_rep = @FactEndDate
      AND b.section_name = N'Срочные'
      AND b.dt_open >= @WeekFrom
      AND b.dt_open <= @WeekTo
    GROUP BY b.cli_id
),
ns_agg AS
(
    SELECT
          b.cli_id
        , SUM(CASE WHEN b.dt_rep = @MondayStart THEN b.out_rub ELSE 0 END) AS ns_start_rub
        , SUM(CASE WHEN b.dt_rep = @FactEndDate THEN b.out_rub ELSE 0 END) AS ns_end_rub
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.section_name = N'Накопительный счёт'
    GROUP BY b.cli_id
)
SELECT
      c.cli_id
    , s.segment_name
    , ISNULL(e.cnt_con_exit, 0) AS cnt_con_exit
    , ISNULL(e.vol_exit_rub, 0) AS vol_exit_rub
    , e.wavg_rate_exit
    , ISNULL(o.cnt_con_open, 0) AS cnt_con_open
    , ISNULL(o.vol_open_rub, 0) AS vol_open_rub
    , o.wavg_rate_open
    , ISNULL(n.ns_start_rub, 0) AS ns_start_rub
    , ISNULL(n.ns_end_rub, 0) AS ns_end_rub
    , ISNULL(n.ns_end_rub, 0) - ISNULL(n.ns_start_rub, 0) AS ns_delta_rub
    , CASE WHEN ISNULL(o.cnt_con_open, 0) > 0 THEN 1 ELSE 0 END AS opened_new_dep_flag
FROM #clients_scope c
LEFT JOIN client_segment s
    ON c.cli_id = s.cli_id
LEFT JOIN exit_agg e
    ON c.cli_id = e.cli_id
LEFT JOIN open_agg o
    ON c.cli_id = o.cli_id
LEFT JOIN ns_agg n
    ON c.cli_id = n.cli_id
ORDER BY vol_exit_rub DESC, c.cli_id;

Если хочешь, я еще сразу соберу третью версию: агрегация уже не по клиентам, а по сегментам ДЧБО / Розничный бизнес.
