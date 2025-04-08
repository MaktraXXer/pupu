-------------------------------------------------------------------------------
-- Ввод дат (при работе в SQL*Plus, SQL Developer и т.д.)
-- ACCEPT p_start_date CHAR PROMPT 'Enter start date (dd.mm.yyyy): '; 
-- ACCEPT p_end_date   CHAR PROMPT 'Enter end date (dd.mm.yyyy): ';
-- DEFINE p_start_date;
-- DEFINE p_end_date;
-------------------------------------------------------------------------------

WITH base_data AS (
    SELECT
        /* Приводим TIMESTAMP WITH TIME ZONE к DATE, чтобы считать «целый день» */
        CAST(d.date_created AS DATE) AS created_date_only,
        
        /* 
          Начиная с этой даты считаем неделю (7 дней), сдвигаясь от p_start_date.
          Форма:
            TRUNC(дата) 
              - MOD( TRUNC(дата) - TRUNC(p_start_date), 7 )
          
          При этом p_start_date тоже переводим к DATE через TO_DATE('&p_start_date','DD.MM.YYYY'). 
          Если нужно «привязаться» строго к самим суткам p_start_date, то TRUNC(...) можно не делать.
        */
        TRUNC(CAST(d.date_created AS DATE)) 
         - MOD(
              TRUNC(CAST(d.date_created AS DATE)) 
              - TRUNC(TO_DATE('&p_start_date','DD.MM.YYYY')),
              7
           ) AS week_start,
        
        d.transfer_amount_value,
        d.client_id
    FROM ods_011.documents d
    WHERE d.transfer_type = 'PHONE'
      AND d.digest LIKE 'Перевод по номеру телефона INTERNAL%'
      AND d.active_flag = 'Y'
      AND d.status = 'Processed'

      -- Фильтруем от 00:00 p_start_date до конца p_end_date (23:59:59.999...)
      AND d.date_created >= CAST(TO_DATE('&p_start_date','DD.MM.YYYY') AS TIMESTAMP)
      AND d.date_created <  CAST(TO_DATE('&p_end_date','DD.MM.YYYY') + 1 AS TIMESTAMP)
),
weekly_agg AS (
    SELECT
        week_start,
        
        /* Формируем текстовый диапазон для поля week: "01 Янв - 07 Янв" и т.п. */
        TO_CHAR(week_start, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
          || ' - ' ||
        TO_CHAR(week_start + 6, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN') AS week,
        
        /* Сумма переводов, где единоразовый перевод < 100 000 руб. */
        SUM(CASE WHEN transfer_amount_value <  100000 THEN transfer_amount_value ELSE 0 END)
          AS sum_trans_lt100k,
        
        /* Уникальные клиенты, для переводов < 100 000 руб. */
        COUNT(DISTINCT CASE WHEN transfer_amount_value < 100000 THEN client_id END)
          AS unique_clients_lt100k,
        
        /* Сумма переводов, где единоразовый перевод >= 100 000 руб. */
        SUM(CASE WHEN transfer_amount_value >= 100000 THEN transfer_amount_value ELSE 0 END)
          AS sum_trans_ge100k,
        
        /* Уникальные клиенты, для переводов >= 100 000 руб. */
        COUNT(DISTINCT CASE WHEN transfer_amount_value >= 100000 THEN client_id END)
          AS unique_clients_ge100k
    FROM base_data
    GROUP BY week_start
)
SELECT
    week_start,
    week,
    sum_trans_lt100k,
    unique_clients_lt100k,
    sum_trans_ge100k,
    unique_clients_ge100k
FROM weekly_agg
ORDER BY week_start;
