-------------------------------------------------------------------------------
-- Ввод дат (при работе в SQL*Plus/SQL Developer и т. п.)
-- ACCEPT p_start_date CHAR PROMPT 'Enter start date (dd.mm.yyyy) [четверг]: ';
-- ACCEPT p_end_date   CHAR PROMPT 'Enter end date (dd.mm.yyyy)   [четверг]: ';
-- DEFINE p_start_date;
-- DEFINE p_end_date;
-------------------------------------------------------------------------------
WITH base_data AS (
    SELECT
        d.date_created,
        d.transfer_amount_value,
        d.client_id
        
    FROM ods_011.documents d
    WHERE d.transfer_type = 'PHONE'
      AND d.digest LIKE 'Перевод по номеру телефона INTERNAL%'
      AND d.active_flag = 'Y'
      AND d.status = 'Processed'
      
      -- Фильтруем записи в интервале [p_start_date 00:00:00; p_end_date 00:00:00)
      AND d.date_created >= FROM_TZ(
                             TO_TIMESTAMP('&p_start_date 00:00:00','DD.MM.YYYY HH24:MI:SS'),
                             'Europe/Moscow'
                           )
      AND d.date_created <  FROM_TZ(
                             TO_TIMESTAMP('&p_end_date 00:00:00','DD.MM.YYYY HH24:MI:SS'),
                             'Europe/Moscow'
                           )
),
calc_week AS (
    SELECT
        /* 
          «Отбрасываем» время/таймзону, получая «чистую» дату, 
          чтобы проще было считать недельные интервалы 
        */
        TRUNC(CAST(d.date_created AT TIME ZONE 'Europe/Moscow' AS DATE)) AS created_date_only,

        /* 
          Вычисляем начало «банковской недели» (четверг), 
          отталкиваясь от p_start_date (которая тоже должна быть четвергом).
          Формула: 
              week_start = created_date_only
                         - MOD( created_date_only - TRUNC(p_start_date), 7 )
          Грубо говоря, это «тот самый четверг, к которому относится created_date_only».
        */
        TRUNC(CAST(d.date_created AT TIME ZONE 'Europe/Moscow' AS DATE))
        - MOD(
             TRUNC(CAST(d.date_created AT TIME ZONE 'Europe/Moscow' AS DATE))
             - TRUNC(TO_DATE('&p_start_date','DD.MM.YYYY')),
             7
          ) AS week_start,

        d.transfer_amount_value,
        d.client_id
    FROM base_data d
),
weekly_agg AS (
    SELECT
        week_start,
        
        /* Текстовый диапазон: «DD Mon - DD Mon» (например, «30 Мар - 05 Апр») */
        TO_CHAR(week_start,       'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
         || ' - ' ||
        TO_CHAR(week_start + 6,   'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN') AS week_range,
        
        /* Сумма переводов < 100 тыс. руб. */
        SUM(CASE WHEN transfer_amount_value <  100000 THEN transfer_amount_value ELSE 0 END)
          AS sum_trans_lt100k,
        
        /* Уникальные клиенты для переводов < 100 тыс. руб. */
        COUNT(DISTINCT CASE WHEN transfer_amount_value < 100000 THEN client_id END)
          AS unique_clients_lt100k,
        
        /* Сумма переводов >= 100 тыс. руб. */
        SUM(CASE WHEN transfer_amount_value >= 100000 THEN transfer_amount_value ELSE 0 END)
          AS sum_trans_ge100k,
        
        /* Уникальные клиенты для переводов >= 100 тыс. руб. */
        COUNT(DISTINCT CASE WHEN transfer_amount_value >= 100000 THEN client_id END)
          AS unique_clients_ge100k
        
    FROM calc_week
    GROUP BY week_start
)
SELECT
    week_start,
    week_range,
    sum_trans_lt100k,
    unique_clients_lt100k,
    sum_trans_ge100k,
    unique_clients_ge100k
FROM weekly_agg
ORDER BY week_start;
