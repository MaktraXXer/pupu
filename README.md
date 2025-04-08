-------------------------------------------------------------------------------
-- Ввод дат
--ACCEPT p_start_date CHAR PROMT 'Enter start date (dd.mm.yyyy): '; 
--ACCEPT p_end_date   CHAR PROMT 'Enter end date (dd.mm.yyyy): ';
--DEFINE p_start_date;
--DEFINE p_end_date;
-------------------------------------------------------------------------------
WITH base_data AS (
  SELECT /*+ parallel(4) */
         -- Определяем начало недели по дате создания документа
         TRUNC(d.created) - MOD(TRUNC(d.created) - TO_DATE(&p_start_date, 'DD.MM.YYYY'), 7) AS week_start,
         d.transfer_amount_value,
         d.client_id
  FROM ods_011.documents d
  WHERE d.transfer_type = 'PHONE'
    AND d.digest LIKE 'Перевод по номеру телефона INTERNAL%'
    AND d.active_flag = 'Y'
    AND d.status = 'Processed'
    AND d.created >= TO_DATE(&p_start_date, 'DD.MM.YYYY')
    AND d.created < TO_DATE(&p_end_date, 'DD.MM.YYYY')
),
weekly_agg AS (
  SELECT
    week_start,
    -- Формируем текстовое поле с диапазоном дат недели (например, 01 Янв - 07 Янв)
    TO_CHAR(week_start, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
         || ' - ' ||
    TO_CHAR(week_start + 6, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN') AS week,
    -- Сумма переводов, где единоразовый перевод < 100 тыс. руб.
    SUM(CASE WHEN transfer_amount_value < 100000 THEN transfer_amount_value ELSE 0 END) AS sum_trans_lt100k,
    -- Количество уникальных клиентов (по client_id) для переводов < 100 тыс. руб.
    COUNT(DISTINCT CASE WHEN transfer_amount_value < 100000 THEN client_id END) AS unique_clients_lt100k,
    -- Сумма переводов, где единоразовый перевод >= 100 тыс. руб.
    SUM(CASE WHEN transfer_amount_value >= 100000 THEN transfer_amount_value ELSE 0 END) AS sum_trans_ge100k,
    -- Количество уникальных клиентов (по client_id) для переводов >= 100 тыс. руб.
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
