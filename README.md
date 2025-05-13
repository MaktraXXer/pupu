USE ALM;
GO
DECLARE @date_prev  date = '2025-02-28';  -- «старая» дата
DECLARE @date_curr  date = '2025-03-31';  -- «новая» дата

/*===== агрегируем баланс на каждую дату =====*/
WITH balance_agg AS (
    SELECT
        dt_rep,
        cli_id,
        SUM(out_rub)/1e6                                           AS total_rub,
        SUM(CASE WHEN section_name = N'Срочные'           THEN out_rub END)/1e6 AS term_rub,
        SUM(CASE WHEN section_name = N'До востребования'  THEN out_rub END)/1e6 AS dvs_rub,
        SUM(CASE WHEN section_name = N'Накопительный счёт' THEN out_rub END)/1e6 AS ns_rub,
        MAX(CASE WHEN TSEGMENTNAME = N'ДЧБО'             THEN 1 ELSE 0 END)     AS DCHBO_flag,
        MAX(CASE WHEN TSEGMENTNAME = N'Розничный бизнес' THEN 1 ELSE 0 END)     AS RB_flag
    FROM  ALM.dbo.balance_rest_all WITH (NOLOCK)
    WHERE od_flag = 1
      AND block_name = N'Привлечение ФЛ'
      AND section_name NOT IN (N'Аккредитивы', N'Брокерское обслуживание',
                               N'Аккредитив под строительство')
      AND dt_rep IN (@date_prev, @date_curr)
    GROUP BY dt_rep, cli_id
),

/*===== ищем новых клиентов: есть @date_curr, нет @date_prev =====*/
new_clients AS (
    SELECT c.*
    FROM   balance_agg              AS c
    LEFT   JOIN balance_agg AS p
           ON  p.cli_id = c.cli_id
           AND p.dt_rep = @date_prev
    WHERE  c.dt_rep = @date_curr
      AND  p.cli_id IS NULL          -- не было в старой дате
)

/*===== итоговая сводка по трём сегментам =====*/
SELECT Segment,
       COUNT(*)                                    AS [Количество клиентов],
       AVG(total_rub)                              AS [средний объём ТОТАЛ],
       AVG(dvs_rub)                                AS [средний объём ДВС],
       AVG(ns_rub)                                 AS [средний объём НС],
       AVG(term_rub)                               AS [средний объём Срочные]
FROM (
    SELECT 'RB_only' AS Segment, *
    FROM   new_clients
    WHERE  RB_flag = 1 AND DCHBO_flag = 0

    UNION ALL
    SELECT 'DCHBO', *
    FROM   new_clients
    WHERE  DCHBO_flag = 1

    UNION ALL
    SELECT 'All_new', *
    FROM   new_clients
) s
GROUP BY Segment
ORDER BY CASE Segment
             WHEN 'RB_only' THEN 1
             WHEN 'DCHBO'  THEN 2
             ELSE 3
         END;
