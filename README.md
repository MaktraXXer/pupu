-- ===================================
-- ОДИН СВОДНЫЙ ЗАПРОС ПО ВСЕМ БАНКАМ
-- ===================================
WITH
-------------------------------------------------------------------------------
-- 1) ВХОДЯЩИЕ ПЕРЕВОДЫ (AGGREGATE) -- суммируем всё по неделям, по всем банкам
-------------------------------------------------------------------------------
incoming_data AS (
    SELECT
        -- Определяем "начало недели" по вашему алгоритму
        TRUNC(doc.dt_receipt)
          - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'), 7)
          AS week_start,

        -- Суммарный объём входящих за эту неделю
        SUM(e.vl_rub) AS incoming_sum_trans,

        -- Уникальные клиенты-получатели (COUNT DISTINCT ID)
        COUNT(DISTINCT doc.cli_id_recip) AS incoming_unique_clients

    FROM dds.doc_order doc
    JOIN dds.entry e
        ON doc.doc_id = e.doc_id
    JOIN dds.client c
        ON c.cli_id = doc.cli_id_recip
    LEFT JOIN dds.client_bank cb
        ON cb.cli_id = doc.cli_id_recip
    LEFT JOIN bic_map bm
        ON bm.bik = doc.payer_bank_bic
       AND bm.rn  = 1

    WHERE
          -- Фильтр по валютам (строго '810')
          e.cur_db = '810'
      AND e.cur_cr = '810'

          -- Тип направления: INGOING
      AND doc.doc_trans_direct = 'INGOING'

          -- Физ. лицо
      AND c.cli_type = 'I'

          -- Исключаем "псевдо-даты" (4444-01-01)
      AND doc.dt_receipt != DATE '4444-01-01'

          -- Период
      AND doc.dt_receipt >= TO_DATE(&p_start_date, 'DD.MM.YYYY')
      AND doc.dt_receipt <  TO_DATE(&p_end_date,   'DD.MM.YYYY')

          -- Cумма не менее 100k
      AND e.vl_rub >= 100000

          -- Исключаем банк "Дом РФ" при необходимости
      AND (   bm.cli_short_name IS NULL
           OR UPPER(bm.cli_short_name) NOT LIKE '%ДОМ%'
          )

          -- Исключаем, где payer/recip = ИП
      AND NOT REGEXP_LIKE( UPPER(doc.payer_name), '(^|\\s)(ИП|IP)(\\s|$)' )
      AND NOT REGEXP_LIKE( UPPER(doc.recip_name), '(^|\\s)(ИП|IP)(\\s|$)' )

          -- "Самому себе": либо одинаковый cli_id, либо INN, либо ФИО
      AND (
             doc.cli_id_payer = doc.cli_id_recip
          OR doc.payer_inn    = doc.recip_inn
          OR (
                REGEXP_REPLACE(UPPER(doc.payer_name),'[^A-ZА-Я]', '')
                = REGEXP_REPLACE(UPPER(doc.recip_name),'[^A-ZА-Я]', '')
             )
      )

          -- Исключаем вариант, что плательщик и получатель в одном банке?
          -- Судя по условию "AND doc.payer_bank_bic <> doc.recip_bank_bic"
      AND doc.payer_bank_bic <> doc.recip_bank_bic

    GROUP BY
        TRUNC(doc.dt_receipt)
         - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'),7)
),

-------------------------------------------------------------------------------
-- 2) ИСХОДЯЩИЕ ПЕРЕВОДЫ (AGGREGATE) -- суммируем всё по неделям, по всем банкам
-------------------------------------------------------------------------------
outgoing_data AS (
    SELECT
        TRUNC(doc.dt_receipt)
          - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'), 7)
          AS week_start,

        SUM(e.vl_rub) AS outgoing_sum_trans,
        COUNT(DISTINCT doc.cli_id_payer) AS outgoing_unique_clients

    FROM dds.doc_order doc
    JOIN dds.entry e
        ON doc.doc_id = e.doc_id
    JOIN dds.client c
        ON c.cli_id = doc.cli_id_payer
    LEFT JOIN dds.client_bank cb
        ON cb.cli_id = doc.cli_id_payer
    LEFT JOIN bic_map bm
        ON bm.bik = doc.recip_bank_bic
       AND bm.rn  = 1

    WHERE
          e.cur_db = '810'
      AND e.cur_cr = '810'
      AND doc.doc_trans_direct = 'OUTGOING'
      AND c.cli_type = 'I'
      AND doc.dt_receipt != DATE '4444-01-01'
      AND doc.dt_receipt >= TO_DATE(&p_start_date,'DD.MM.YYYY')
      AND doc.dt_receipt <  TO_DATE(&p_end_date,  'DD.MM.YYYY')
      AND e.vl_rub >= 100000
      AND (   bm.cli_short_name IS NULL
           OR UPPER(bm.cli_short_name) NOT LIKE '%ДОМ%'
          )
      AND NOT REGEXP_LIKE(UPPER(doc.payer_name), '(^|\\s)(ИП|IP)(\\s|$)' )
      AND NOT REGEXP_LIKE(UPPER(doc.recip_name), '(^|\\s)(ИП|IP)(\\s|$)' )
      AND (
             doc.cli_id_payer = doc.cli_id_recip
          OR doc.payer_inn    = doc.recip_inn
          OR (
               REGEXP_REPLACE(UPPER(doc.payer_name),'[^A-ZА-Я]', '')
               = REGEXP_REPLACE(UPPER(doc.recip_name),'[^A-ZА-Я]', '')
             )
      )
      AND doc.payer_bank_bic <> doc.recip_bank_bic

    GROUP BY
        TRUNC(doc.dt_receipt)
         - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'),7)
),

-------------------------------------------------------------------------------
-- 3) ОТБОР КЛИЕНТОВ, у которых были И входящие, И исходящие в одной неделе
--    (по всем банкам сразу, реально уникальные ID)
-------------------------------------------------------------------------------
both_in_out AS (
    SELECT
      i.week_start,
      COUNT(DISTINCT i.client_id) AS both_directions_unique_clients
    FROM (
      -- INCOMING clients
      SELECT
        TRUNC(doc.dt_receipt)
          - MOD(TRUNC(doc.dt_receipt)-TO_DATE(&p_start_date,'DD.MM.YYYY'),7)
          AS week_start,
        doc.cli_id_recip AS client_id
      FROM dds.doc_order doc
      JOIN dds.entry e ON doc.doc_id = e.doc_id
      WHERE
            e.cur_db='810'
        AND e.cur_cr='810'
        AND doc.doc_trans_direct='INGOING'
        AND doc.dt_receipt >= TO_DATE(&p_start_date,'DD.MM.YYYY')
        AND doc.dt_receipt <  TO_DATE(&p_end_date,'DD.MM.YYYY')
        AND e.vl_rub>=100000
        AND doc.payer_bank_bic <> doc.recip_bank_bic
        -- Плюс те же ограничения (ИП и т.д.) если нужно --
    ) i
    JOIN (
      -- OUTGOING clients
      SELECT
        TRUNC(doc.dt_receipt)
          - MOD(TRUNC(doc.dt_receipt)-TO_DATE(&p_start_date,'DD.MM.YYYY'),7)
          AS week_start,
        doc.cli_id_payer AS client_id
      FROM dds.doc_order doc
      JOIN dds.entry e ON doc.doc_id = e.doc_id
      WHERE
            e.cur_db='810'
        AND e.cur_cr='810'
        AND doc.doc_trans_direct='OUTGOING'
        AND doc.dt_receipt >= TO_DATE(&p_start_date,'DD.MM.YYYY')
        AND doc.dt_receipt <  TO_DATE(&p_end_date,'DD.MM.YYYY')
        AND e.vl_rub>=100000
        AND doc.payer_bank_bic <> doc.recip_bank_bic
        -- Плюс те же ограничения (ИП и т.д.) если нужно --
    ) o
      ON  i.week_start = o.week_start
      AND i.client_id  = o.client_id
    GROUP BY i.week_start
),

-------------------------------------------------------------------------------
-- 4) ФИНАЛЬНОЕ СЛИЯНИЕ -- week_start + входящие + исходящие + оба
-------------------------------------------------------------------------------
final_agg AS (
  SELECT
     -- Неделя (понедельник)
     COALESCE(i.week_start, o.week_start) AS week_start,

     -- Входящие
     NVL(i.incoming_sum_trans, 0)      AS incoming_sum_trans,
     NVL(i.incoming_unique_clients, 0) AS incoming_unique_clients,

     -- Исходящие
     NVL(o.outgoing_sum_trans, 0)      AS outgoing_sum_trans,
     NVL(o.outgoing_unique_clients, 0) AS outgoing_unique_clients,

     -- Оба направления
     NVL(b.both_directions_unique_clients, 0) AS both_in_out_clients

  FROM incoming_data i
  FULL OUTER JOIN outgoing_data o
     ON i.week_start = o.week_start
  LEFT JOIN both_in_out b
     ON COALESCE(i.week_start, o.week_start) = b.week_start
)

-- Теперь выбираем результат, сортируя по неделям
SELECT
  fa.week_start,
  -- Для удобного чтения: "01.Jan - 07.Jan"
  TO_CHAR(fa.week_start, 'DD.Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
    || ' - '
    || TO_CHAR(fa.week_start + 6, 'DD.Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
    AS week_range,

  fa.incoming_sum_trans,
  fa.incoming_unique_clients,
  fa.outgoing_sum_trans,
  fa.outgoing_unique_clients,
  fa.both_in_out_clients

FROM final_agg fa
ORDER BY fa.week_start
;
