-------------------------------------------------------------------------------
-- Ввод дат (для SQL*Plus)
--ACCEPT p_start_date CHAR PROMPT 'Enter start date (dd.mm.yyyy): '; 
--ACCEPT p_end_date   CHAR PROMPT 'Enter end date (dd.mm.yyyy): ';
--DEFINE p_start_date;
--DEFINE p_end_date;
-------------------------------------------------------------------------------
WITH base_data AS (
  SELECT /*+ parallel(4) */
         -- Приводим поле date_created к нужной временной зоне и вычисляем начало недели.
         TRUNC(d.date_created AT TIME ZONE 'Europe/Moscow') 
           - MOD(
                TRUNC(d.date_created AT TIME ZONE 'Europe/Moscow')
                - TRUNC( 
                     FROM_TZ(TO_TIMESTAMP('&p_start_date 00:00:00', 'DD.MM.YYYY HH24:MI:SS'), 'Europe/Moscow')
                     ),
                7
             ) AS week_start,
         d.transfer_amount_value,
         d.client_id,
         d.date_created
  FROM ods_011.documents d
  WHERE d.transfer_type = 'PHONE'
    AND d.digest LIKE 'Перевод по номеру телефона INTERNAL%'
    AND d.active_flag = 'Y'
    AND d.status = 'Processed'
    -- Фильтрация по периоду: от начала дня p_start_date до конца дня p_end_date.
    AND d.date_created >= FROM_TZ(TO_TIMESTAMP('&p_start_date 00:00:00', 'DD.MM.YYYY HH24:MI:SS'), 'Europe/Moscow')
    AND d.date_created <= FROM_TZ(TO_TIMESTAMP('&p_end_date 23:59:59.999999', 'DD.MM.YYYY HH24:MI:SS.FF6'), 'Europe/Moscow')
),
weekly_agg AS (
  SELECT
    week_start,
    -- Формируем текстовый диапазон дат недели (например, "01 Янв - 07 Янв")
    TO_CHAR(week_start, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
      || ' - ' ||
    TO_CHAR(week_start + 6, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN') AS week,
    -- Сумма переводов, где единоразовый перевод < 100 тыс. руб.
    SUM(CASE WHEN transfer_amount_value < 100000 THEN transfer_amount_value ELSE 0 END) AS sum_trans_lt100k,
    -- Количество уникальных клиентов для переводов < 100 тыс. руб.
    COUNT(DISTINCT CASE WHEN transfer_amount_value < 100000 THEN client_id END) AS unique_clients_lt100k,
    -- Сумма переводов, где единоразовый перевод >= 100 тыс. руб.
    SUM(CASE WHEN transfer_amount_value >= 100000 THEN transfer_amount_value ELSE 0 END) AS sum_trans_ge100k,
    -- Количество уникальных клиентов для переводов >= 100 тыс. руб.
    COUNT(DISTINCT CASE WHEN transfer_amount_value >= 100000 THEN client_id END) AS unique_clients_ge100k
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
