WITH
-------------------------------------------------------------------------------
-- 0) "Справочник BIC -> cli_short_name" (без изменений)
-------------------------------------------------------------------------------
bic_map AS (
    SELECT
        cb.bik,
        c.cli_short_name,
        ROW_NUMBER() OVER (PARTITION BY cb.bik ORDER BY cb.cli_id) AS rn
    FROM dds.client_bank cb
    JOIN dds.client c
        ON c.cli_id = cb.cli_id
),

-------------------------------------------------------------------------------
-- 1A) Деталь по входящим переводам (разбивка по банкам)
-------------------------------------------------------------------------------
incoming_base_data AS (
    SELECT
        TRUNC(doc.dt_receipt)
         - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'), 7)
         AS week_start,

        CASE
            WHEN UPPER(bm.cli_short_name) LIKE '%СБЕРБАНК%' THEN 'Сбербанк'
            WHEN UPPER(bm.cli_short_name) LIKE '%ВБРР%'     THEN 'Банк ВБРР'
            WHEN UPPER(bm.cli_short_name) LIKE '%ВТБ%'      THEN 'ВТБ'
            WHEN    UPPER(bm.cli_short_name) LIKE '%АЛЬФА%-БАНК%'
                 OR UPPER(bm.cli_short_name) LIKE '%АЛЬФА БАНК%'
                 THEN 'Альфа Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%АК БАРС%'  THEN 'Ак Барс Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%УРАЛСИБ%'  THEN 'БАНК УРАЛСИБ'
            WHEN    UPPER(bm.cli_short_name) LIKE '%ГАЗПРОМБАНК%'
                 OR UPPER(bm.cli_short_name) LIKE '%ГПБ%'
                 THEN 'Газпромбанк'
            WHEN    UPPER(bm.cli_short_name) LIKE '%ПРОМСВЯЗЬБАНК%'
                 OR UPPER(bm.cli_short_name) LIKE '%ПАО "ПРОМСВЯЗЬБАНК"%'
                 THEN 'Промсвязьбанк'
            WHEN UPPER(bm.cli_short_name) LIKE '%РОСБАНК%'  THEN 'Росбанк'
            WHEN UPPER(bm.cli_short_name) LIKE '%РНКБ%'     THEN 'РНКБ Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%РАЙФФАЙЗЕН%'
                 THEN 'Райффайзен Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%РОССЕЛЬХОЗ%'
                 THEN 'Россельхозбанк'
            WHEN UPPER(bm.cli_short_name) LIKE '%СОВКОМБАНК%'
                 THEN 'Совкомбанк'
            WHEN    UPPER(bm.cli_short_name) LIKE '%МОСКОВСКИЙ КРЕДИТНЫЙ БАНК%'
                 OR UPPER(bm.cli_short_name) LIKE '%МКБ%'
                 THEN 'МКБ (Московский кредитный банк)'
            WHEN UPPER(bm.cli_short_name) LIKE '%ЮНИКРЕДИТ%'
                 THEN 'ЮниКредит Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%ЭКСПОБАНК%'
                 THEN 'Экспобанк'
            WHEN UPPER(bm.cli_short_name) LIKE '%ОТП БАНК%'
                 THEN 'ОТП Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%ОЗОН БАНК%'
                 THEN 'Озон Банк (Ozon)'
            WHEN UPPER(bm.cli_short_name) LIKE '%ПОЧТА БАНК%'
                 THEN 'Почта Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%НБД%-БАНК%'
                 THEN 'НБД-Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%СДМ%-БАНК%'
                 THEN 'СДМ-Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%ЛОКО%-БАНК%'
                 THEN 'КБ ЛОКО-Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%УБРИР%'
                 THEN 'УБРиР'
            WHEN UPPER(bm.cli_short_name) LIKE '%УРАЛ ФД%'
                 THEN 'Урал ФД'
            WHEN UPPER(bm.cli_short_name) LIKE '%УРАЛПРОМБАНК%'
                 THEN 'АО "УРАЛПРОМБАНК"'
            WHEN UPPER(bm.cli_short_name) LIKE '%АБСОЛЮТ%'
                 THEN 'Абсолют Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%РУССКИЙ СТАНДАРТ%'
                 THEN 'Банк Русский Стандарт'
            WHEN UPPER(bm.cli_short_name) LIKE '%БАНК СИНАРА%'
                 THEN 'Банк Синара'
            WHEN UPPER(bm.cli_short_name) LIKE '%МТС%-БАНК%'
                 THEN 'МТС-Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%БАНК ЗЕНИТ%'
                 THEN 'Банк ЗЕНИТ'
            WHEN UPPER(bm.cli_short_name) LIKE '%МОДУЛЬБАНК%'
                 THEN 'КБ Модульбанк'
            WHEN UPPER(bm.cli_short_name) LIKE '%ФК ОТКРЫТИЕ%'
                 THEN 'Банк ФК Открытие'
            WHEN UPPER(bm.cli_short_name) LIKE '%КРЕДИТ ЕВРОПА%'
                 THEN 'Кредит Европа Банк (Россия)'
            WHEN UPPER(bm.cli_short_name) LIKE '%АЗИАТСКО%-ТИХООКЕАНСКИЙ%'
                 THEN 'Азиатско-Тихоокеанский Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%ББР БАНК%'
                 THEN 'ББР Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%ТАВРИЧЕСКИЙ%'
                 THEN 'Таврический Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%ИНГОССТРАХ%'
                 THEN 'Ингосстрах Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%ХОУМ КРЕДИТ%'
                 THEN 'Хоум Кредит Банк (Хоум Банк)'
            WHEN UPPER(bm.cli_short_name) LIKE '%КУБАНЬ КРЕДИТ%'
                 THEN 'Кубань Кредит'
            WHEN UPPER(bm.cli_short_name) LIKE '%МЕТАЛЛИНВЕСТБАНК%'
                 THEN 'Металлинвестбанк'
            WHEN UPPER(bm.cli_short_name) LIKE '%НОВИКОМБАНК%'
                 THEN 'НОВИКОМБАНК'
            WHEN UPPER(bm.cli_short_name) LIKE '%ТИНЬКОФФ%'
                 THEN 'Тинькофф Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%РОСТФИНАНС%'
                 THEN 'КБ  РостФинанс'
            WHEN UPPER(bm.cli_short_name) LIKE '%СОЛИДАРНОСТ%'
                 THEN 'КБ Солидарность'
            WHEN UPPER(bm.cli_short_name) LIKE '%ЦИФРА БАНК%'
                 THEN 'ФФИН Банк (Цифра банк)'
            WHEN UPPER(bm.cli_short_name) LIKE '%ТАТСОЦБАНК%'
                 THEN 'ТАТСОЦБАНК'
            WHEN UPPER(bm.cli_short_name) LIKE '%ТРАНСКАПИТАЛБАНК%'
                 THEN 'ТРАНСКАПИТАЛБАНК'
            WHEN UPPER(bm.cli_short_name) LIKE '%ТОЧКА%'
                 THEN 'Точка Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%СВОЙ БАНК%'
                 THEN 'Свой Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%СИТИБАНК%'
                 THEN 'Ситибанк'
            WHEN UPPER(bm.cli_short_name) LIKE '%РЕНЕССАНС%'
                 THEN 'Ренессанс Кредит'
            WHEN UPPER(bm.cli_short_name) LIKE '%ДАЛЬНЕВОСТОЧНЫЙ БАНК%'
                 THEN 'Дальневосточный банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%ЗОЛОТАЯ КОРОНА%'
                 THEN 'Золотая Корона - KoronaPay (РНКО Платежный Центр)'
            WHEN UPPER(bm.cli_short_name) LIKE '%ЯНДЕКС БАНК%'
                 THEN 'Яндекс Банк'
            WHEN UPPER(bm.cli_short_name) LIKE '%АБ "РОССИЯ"%'
                 THEN 'АБ РОССИЯ'
            ELSE bm.cli_short_name
        END AS bank_name,

        SUM(e.vl_rub) AS incoming_sum_trans,
        COUNT(DISTINCT doc.cli_id_recip) AS incoming_unique_clients
    FROM dds.doc_order doc
    JOIN dds.entry e
        ON doc.doc_id = e.doc_id
    JOIN dds.client c
        ON doc.cli_id_recip = c.cli_id
    LEFT JOIN bic_map bm
        ON bm.bik = doc.payer_bank_bic
       AND bm.rn  = 1
    WHERE
          -- cur_db/cur_cr = 810, doc_trans_direct='INGOING', клиенты типа 'I' --
          e.cur_db = '810'
      AND e.cur_cr = '810'
      AND doc.doc_trans_direct = 'INGOING'
      AND c.cli_type = 'I'
      AND doc.payer_bank_bic <> doc.recip_bank_bic
      AND doc.dt_receipt != DATE '4444-01-01'
      AND doc.dt_receipt >= TO_DATE(&p_start_date, 'DD.MM.YYYY')
      AND doc.dt_receipt <  TO_DATE(&p_end_date,   'DD.MM.YYYY')
      AND e.vl_rub >= 100000
      -- Исключаем, где payer_name/recip_name = ИП, + проверяем inn, etc. --
      AND NOT REGEXP_LIKE( UPPER(doc.payer_name), '(^|\\s)(ИП|IP)(\\s|$)' )
      AND NOT REGEXP_LIKE( UPPER(doc.recip_name), '(^|\\s)(ИП|IP)(\\s|$)' )
      AND (
            doc.cli_id_payer = doc.cli_id_recip
         OR doc.payer_inn    = doc.recip_inn
         OR ( REGEXP_REPLACE(UPPER(doc.payer_name), '[^A-ZА-Я]', '')
            = REGEXP_REPLACE(UPPER(doc.recip_name), '[^A-ZА-Я]', '') )
      )
      AND UPPER(bm.cli_short_name) NOT LIKE '%ДОМ%'
    GROUP BY
        TRUNC(doc.dt_receipt)
         - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'), 7),
        CASE
         -- (тот же CASE для bank_name, чтобы группировка совпала)
         WHEN UPPER(bm.cli_short_name) LIKE '%СБЕРБАНК%' THEN 'Сбербанк'
         WHEN UPPER(bm.cli_short_name) LIKE '%ВБРР%'     THEN 'Банк ВБРР'
         WHEN UPPER(bm.cli_short_name) LIKE '%ВТБ%'      THEN 'ВТБ'
         ...
         WHEN UPPER(bm.cli_short_name) LIKE '%АБ "РОССИЯ"%'
              THEN 'АБ РОССИЯ'
         ELSE bm.cli_short_name
        END
),

-------------------------------------------------------------------------------
-- 1B) Деталь по исходящим переводам (разбивка по банкам)
-------------------------------------------------------------------------------
outgoing_base_data AS (
    SELECT
        TRUNC(doc.dt_receipt)
         - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'), 7)
         AS week_start,

        CASE
         WHEN UPPER(bm.cli_short_name) LIKE '%СБЕРБАНК%' THEN 'Сбербанк'
         WHEN UPPER(bm.cli_short_name) LIKE '%ВБРР%'     THEN 'Банк ВБРР'
         WHEN UPPER(bm.cli_short_name) LIKE '%ВТБ%'      THEN 'ВТБ'
         ...
         WHEN UPPER(bm.cli_short_name) LIKE '%АБ "РОССИЯ"%'
              THEN 'АБ РОССИЯ'
         ELSE bm.cli_short_name
        END AS bank_name,

        SUM(e.vl_rub) AS outgoing_sum_trans,
        COUNT(DISTINCT doc.cli_id_payer) AS outgoing_unique_clients
    FROM dds.doc_order doc
    JOIN dds.entry e
        ON doc.doc_id = e.doc_id
    JOIN dds.client c
        ON doc.cli_id_payer = c.cli_id
    LEFT JOIN bic_map bm
        ON bm.bik = doc.recip_bank_bic
       AND bm.rn  = 1
    WHERE
          e.cur_db = '810'
      AND e.cur_cr = '810'
      AND doc.doc_trans_direct = 'OUTGOING'
      AND c.cli_type = 'I'
      AND doc.payer_bank_bic <> doc.recip_bank_bic
      AND doc.dt_receipt != DATE '4444-01-01'
      AND doc.dt_receipt >= TO_DATE(&p_start_date, 'DD.MM.YYYY')
      AND doc.dt_receipt <  TO_DATE(&p_end_date,   'DD.MM.YYYY')
      AND e.vl_rub >= 100000
      AND NOT REGEXP_LIKE( UPPER(doc.payer_name), '(^|\\s)(ИП|IP)(\\s|$)' )
      AND NOT REGEXP_LIKE( UPPER(doc.recip_name), '(^|\\s)(ИП|IP)(\\s|$)' )
      AND (
            doc.cli_id_payer = doc.cli_id_recip
         OR doc.payer_inn    = doc.recip_inn
         OR ( REGEXP_REPLACE(UPPER(doc.payer_name), '[^A-ZА-Я]', '')
            = REGEXP_REPLACE(UPPER(doc.recip_name), '[^A-ZА-Я]', '') )
      )
    GROUP BY
        TRUNC(doc.dt_receipt)
         - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'), 7),
        CASE
         WHEN UPPER(bm.cli_short_name) LIKE '%СБЕРБАНК%' THEN 'Сбербанк'
         WHEN UPPER(bm.cli_short_name) LIKE '%ВБРР%'     THEN 'Банк ВБРР'
         WHEN UPPER(bm.cli_short_name) LIKE '%ВТБ%'      THEN 'ВТБ'
         ...
         ELSE bm.cli_short_name
        END
),

-------------------------------------------------------------------------------
-- 1C) Сколько клиентов в конкретный week_start и конкретном банке делали И ВХОД, И ВЫХОД
-------------------------------------------------------------------------------
both_clients AS (
    SELECT
        i.week_start,
        i.bank_name,
        COUNT(DISTINCT i.client_id) AS both_directions_unique_clients
    FROM
    (
      -- те, кто INCOMING
      SELECT
         TRUNC(doc.dt_receipt)
          - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'),7)
          AS week_start,
         CASE
          WHEN UPPER(bm.cli_short_name) LIKE '%СБЕРБАНК%' THEN 'Сбербанк'
          ...
          ELSE bm.cli_short_name
         END AS bank_name,
         doc.cli_id_recip AS client_id
      FROM dds.doc_order doc
      JOIN dds.entry e
         ON doc.doc_id = e.doc_id
      JOIN dds.client c
         ON doc.cli_id_recip = c.cli_id
      LEFT JOIN bic_map bm
         ON bm.bik = doc.payer_bank_bic AND bm.rn=1
      WHERE
            e.cur_db='810'
        AND e.cur_cr='810'
        AND doc.doc_trans_direct='INGOING'
        AND c.cli_type='I'
        AND doc.payer_bank_bic <> doc.recip_bank_bic
        AND doc.dt_receipt >= TO_DATE(&p_start_date,'DD.MM.YYYY')
        AND doc.dt_receipt <  TO_DATE(&p_end_date,'DD.MM.YYYY')
        AND e.vl_rub>=100000
        AND NOT REGEXP_LIKE( UPPER(doc.payer_name),'(^|\\s)(ИП|IP)(\\s|$)' )
        AND NOT REGEXP_LIKE( UPPER(doc.recip_name),'(^|\\s)(ИП|IP)(\\s|$)' )
        AND (
             doc.cli_id_payer=doc.cli_id_recip
          OR doc.payer_inn   = doc.recip_inn
          OR (
               REGEXP_REPLACE(UPPER(doc.payer_name), '[^A-ZА-Я]', '')
               = REGEXP_REPLACE(UPPER(doc.recip_name), '[^A-ZА-Я]', '')
             )
        )
    ) i
    JOIN
    (
      -- те, кто OUTGOING
      SELECT
         TRUNC(doc.dt_receipt)
          - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'),7)
          AS week_start,
         CASE
           WHEN UPPER(bm.cli_short_name) LIKE '%СБЕРБАНК%' THEN 'Сбербанк'
           ...
           ELSE bm.cli_short_name
         END AS bank_name,
         doc.cli_id_payer AS client_id
      FROM dds.doc_order doc
      JOIN dds.entry e
         ON doc.doc_id = e.doc_id
      JOIN dds.client c
         ON doc.cli_id_payer = c.cli_id
      LEFT JOIN bic_map bm
         ON bm.bik = doc.recip_bank_bic AND bm.rn=1
      WHERE
            e.cur_db='810'
        AND e.cur_cr='810'
        AND doc.doc_trans_direct='OUTGOING'
        AND c.cli_type='I'
        AND doc.payer_bank_bic <> doc.recip_bank_bic
        AND doc.dt_receipt >= TO_DATE(&p_start_date,'DD.MM.YYYY')
        AND doc.dt_receipt <  TO_DATE(&p_end_date,'DD.MM.YYYY')
        AND e.vl_rub>=100000
        AND NOT REGEXP_LIKE( UPPER(doc.payer_name),'(^|\\s)(ИП|IP)(\\s|$)' )
        AND NOT REGEXP_LIKE( UPPER(doc.recip_name),'(^|\\s)(ИП|IP)(\\s|$)' )
        AND (
             doc.cli_id_payer=doc.cli_id_recip
          OR doc.payer_inn   = doc.recip_inn
          OR (
               REGEXP_REPLACE(UPPER(doc.payer_name), '[^A-ZА-Я]', '')
               = REGEXP_REPLACE(UPPER(doc.recip_name), '[^A-ZА-Я]', '')
             )
        )
    ) o
      ON  i.week_start = o.week_start
      AND i.bank_name  = o.bank_name
      AND i.client_id  = o.client_id
    GROUP BY i.week_start, i.bank_name
),

-------------------------------------------------------------------------------
-- 2) Собираем деталь (по банкам) во "final_data_by_bank"
-------------------------------------------------------------------------------
final_data_by_bank AS (
    SELECT
      COALESCE(ib.week_start, ob.week_start)       AS week_start,
      COALESCE(ib.bank_name,  ob.bank_name)        AS bank_name,

      NVL(ib.incoming_sum_trans, 0)      AS incoming_sum_trans,
      NVL(ib.incoming_unique_clients, 0) AS incoming_unique_clients,

      NVL(ob.outgoing_sum_trans, 0)      AS outgoing_sum_trans,
      NVL(ob.outgoing_unique_clients, 0) AS outgoing_unique_clients,

      NVL(bc.both_directions_unique_clients, 0) AS both_in_out_clients
    FROM incoming_base_data ib
    FULL OUTER JOIN outgoing_base_data ob
       ON  ib.week_start = ob.week_start
       AND ib.bank_name  = ob.bank_name
    LEFT JOIN both_clients bc
       ON  COALESCE(ib.week_start, ob.week_start) = bc.week_start
       AND COALESCE(ib.bank_name,  ob.bank_name)  = bc.bank_name
),

-------------------------------------------------------------------------------
-- 3A) Агрегат по всем банкам для INCOMING / OUTGOING / BOTH
--     (чтобы получить реально уникальных клиентов по всей выборке)
-------------------------------------------------------------------------------
agg_in AS (
    SELECT
       TRUNC(doc.dt_receipt)
         - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'), 7)
         AS week_start,
       SUM(e.vl_rub)                      AS incoming_sum_trans,
       COUNT(DISTINCT doc.cli_id_recip)   AS incoming_unique_clients
    FROM dds.doc_order doc
    JOIN dds.entry e
       ON doc.doc_id = e.doc_id
    JOIN dds.client c
       ON doc.cli_id_recip = c.cli_id
    WHERE
          e.cur_db='810'
      AND e.cur_cr='810'
      AND doc.doc_trans_direct='INGOING'
      AND c.cli_type='I'
      AND doc.payer_bank_bic <> doc.recip_bank_bic
      AND doc.dt_receipt != DATE '4444-01-01'
      AND doc.dt_receipt >= TO_DATE(&p_start_date,'DD.MM.YYYY')
      AND doc.dt_receipt <  TO_DATE(&p_end_date,'DD.MM.YYYY')
      AND e.vl_rub>=100000
      AND NOT REGEXP_LIKE(UPPER(doc.payer_name),'(^|\\s)(ИП|IP)(\\s|$)' )
      AND NOT REGEXP_LIKE(UPPER(doc.recip_name),'(^|\\s)(ИП|IP)(\\s|$)' )
      AND (
           doc.cli_id_payer=doc.cli_id_recip
        OR doc.payer_inn   = doc.recip_inn
        OR (
             REGEXP_REPLACE(UPPER(doc.payer_name), '[^A-ZА-Я]', '')
             = REGEXP_REPLACE(UPPER(doc.recip_name), '[^A-ZА-Я]', '')
           )
      )
    GROUP BY
       TRUNC(doc.dt_receipt)
        - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'), 7)
),
agg_out AS (
    SELECT
       TRUNC(doc.dt_receipt)
         - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'), 7)
         AS week_start,
       SUM(e.vl_rub)                     AS outgoing_sum_trans,
       COUNT(DISTINCT doc.cli_id_payer)  AS outgoing_unique_clients
    FROM dds.doc_order doc
    JOIN dds.entry e
       ON doc.doc_id = e.doc_id
    JOIN dds.client c
       ON doc.cli_id_payer = c.cli_id
    WHERE
          e.cur_db='810'
      AND e.cur_cr='810'
      AND doc.doc_trans_direct='OUTGOING'
      AND c.cli_type='I'
      AND doc.payer_bank_bic <> doc.recip_bank_bic
      AND doc.dt_receipt != DATE '4444-01-01'
      AND doc.dt_receipt >= TO_DATE(&p_start_date,'DD.MM.YYYY')
      AND doc.dt_receipt <  TO_DATE(&p_end_date,'DD.MM.YYYY')
      AND e.vl_rub>=100000
      AND NOT REGEXP_LIKE(UPPER(doc.payer_name),'(^|\\s)(ИП|IP)(\\s|$)' )
      AND NOT REGEXP_LIKE(UPPER(doc.recip_name),'(^|\\s)(ИП|IP)(\\s|$)' )
      AND (
           doc.cli_id_payer=doc.cli_id_recip
        OR doc.payer_inn   = doc.recip_inn
        OR (
             REGEXP_REPLACE(UPPER(doc.payer_name), '[^A-ZА-Я]', '')
             = REGEXP_REPLACE(UPPER(doc.recip_name), '[^A-ZА-Я]', '')
           )
      )
    GROUP BY
       TRUNC(doc.dt_receipt)
        - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date, 'DD.MM.YYYY'), 7)
),
agg_both AS (
    SELECT
       i.week_start,
       COUNT(DISTINCT i.client_id) AS both_in_out_clients
    FROM
    (
      SELECT
        TRUNC(doc.dt_receipt)
         - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date,'DD.MM.YYYY'), 7)
         AS week_start,
        doc.cli_id_recip AS client_id
      FROM dds.doc_order doc
      JOIN dds.entry e
         ON doc.doc_id = e.doc_id
      WHERE
            e.cur_db='810'
        AND e.cur_cr='810'
        AND doc.doc_trans_direct='INGOING'
        AND doc.dt_receipt>=TO_DATE(&p_start_date,'DD.MM.YYYY')
        AND doc.dt_receipt< TO_DATE(&p_end_date,'DD.MM.YYYY')
        AND e.vl_rub>=100000
        AND doc.payer_bank_bic<>doc.recip_bank_bic
        -- Прочие те же условия для INCOMING
    ) i
    JOIN
    (
      SELECT
        TRUNC(doc.dt_receipt)
         - MOD(TRUNC(doc.dt_receipt) - TO_DATE(&p_start_date,'DD.MM.YYYY'), 7)
         AS week_start,
        doc.cli_id_payer AS client_id
      FROM dds.doc_order doc
      JOIN dds.entry e
         ON doc.doc_id = e.doc_id
      WHERE
            e.cur_db='810'
        AND e.cur_cr='810'
        AND doc.doc_trans_direct='OUTGOING'
        AND doc.dt_receipt>=TO_DATE(&p_start_date,'DD.MM.YYYY')
        AND doc.dt_receipt< TO_DATE(&p_end_date,'DD.MM.YYYY')
        AND e.vl_rub>=100000
        AND doc.payer_bank_bic<>doc.recip_bank_bic
        -- Прочие те же условия для OUTGOING
    ) o
       ON  i.week_start=o.week_start
       AND i.client_id=o.client_id
    GROUP BY i.week_start
),

-------------------------------------------------------------------------------
-- 3B) Собираем агрегатную финальную часть (по всем банкам)
-------------------------------------------------------------------------------
agg_final AS (
    SELECT
       COALESCE(a.week_start, b.week_start) AS week_start,
       NVL(a.incoming_sum_trans,  0) AS incoming_sum_trans,
       NVL(a.incoming_unique_clients,0) AS incoming_unique_clients,
       NVL(b.outgoing_sum_trans,  0) AS outgoing_sum_trans,
       NVL(b.outgoing_unique_clients,0) AS outgoing_unique_clients,
       NVL(c.both_in_out_clients, 0) AS both_in_out_clients
    FROM agg_in a
    FULL OUTER JOIN agg_out b
       ON a.week_start = b.week_start
    LEFT JOIN agg_both c
       ON COALESCE(a.week_start,b.week_start) = c.week_start
),

-------------------------------------------------------------------------------
-- 4) LAG'и для агрегата, чтобы считать WTW и MTM
-------------------------------------------------------------------------------
agg_with_lags AS (
  SELECT
    f.*,

    LAG(f.incoming_sum_trans,1) OVER(ORDER BY f.week_start)   AS inc_sum_prev_week,
    LAG(f.outgoing_sum_trans,1) OVER(ORDER BY f.week_start)   AS out_sum_prev_week,
    LAG(f.incoming_unique_clients,1) OVER(ORDER BY f.week_start) AS inc_ucl_prev_week,
    LAG(f.outgoing_unique_clients,1) OVER(ORDER BY f.week_start) AS out_ucl_prev_week,

    LAG(f.incoming_sum_trans,4) OVER(ORDER BY f.week_start)   AS inc_sum_prev_month,
    LAG(f.outgoing_sum_trans,4) OVER(ORDER BY f.week_start)   AS out_sum_prev_month,
    LAG(f.incoming_unique_clients,4) OVER(ORDER BY f.week_start) AS inc_ucl_prev_month,
    LAG(f.outgoing_unique_clients,4) OVER(ORDER BY f.week_start) AS out_ucl_prev_month
  FROM agg_final f
),

-------------------------------------------------------------------------------
-- 5) Финальный UNION ALL: DETAIL + SUMMARY
-------------------------------------------------------------------------------
union_result AS (
  -------------------------------------------------------------------------
  -- 5A) DETAIL-блок (по неделям, по банкам) + отдельная строка "ALL BANKS"
  -------------------------------------------------------------------------
  SELECT
    'DETAIL' AS block_type,
    d.week_start,
    -- Человеко-читаемый диапазон
    TO_CHAR(d.week_start, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
      || ' - '
      || TO_CHAR(d.week_start+6, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
      AS week,
    d.bank_name,
    d.incoming_sum_trans,
    d.incoming_unique_clients,
    d.outgoing_sum_trans,
    d.outgoing_unique_clients,
    d.both_in_out_clients,
    /* Колонки для SUMMARY нам тут не нужны, но чтобы совпал UNION-формат, поставим NULL */
    CAST(NULL AS VARCHAR2(100)) AS measure,
    CAST(NULL AS NUMBER(18,2))  AS value,
    CAST(NULL AS NUMBER(18,2))  AS wtw,
    CAST(NULL AS NUMBER(18,2))  AS mtm
  FROM final_data_by_bank d

  UNION ALL
  -- Ещё одна часть DETAIL: "ALL BANKS" на каждую неделю
  SELECT
    'DETAIL' AS block_type,
    a.week_start,
    TO_CHAR(a.week_start,'FMDD Mon','NLS_DATE_LANGUAGE=RUSSIAN')
     || ' - ' 
     || TO_CHAR(a.week_start+6,'FMDD Mon','NLS_DATE_LANGUAGE=RUSSIAN'),
    'ALL BANKS' AS bank_name,
    a.incoming_sum_trans,
    a.incoming_unique_clients,
    a.outgoing_sum_trans,
    a.outgoing_unique_clients,
    a.both_in_out_clients,
    NULL, NULL, NULL, NULL
  FROM agg_final a

  -------------------------------------------------------------------------
  -- 5B) SUMMARY-блок -- только по последней неделе, с приращениями
  -------------------------------------------------------------------------
  UNION ALL
  -- Примерно как в вашем запросе со спб C2C
  SELECT
    'SUMMARY',
    x.week_start,
    TO_CHAR(x.week_start, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
      || ' - '
      || TO_CHAR(x.week_start+6, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN'),
    NULL AS bank_name,
    NULL, NULL, NULL, NULL, NULL,
    'Уникальные клиенты (вход.)' AS measure,
    x.incoming_unique_clients    AS value,
    CASE WHEN x.inc_ucl_prev_week IS NULL THEN NULL
         ELSE x.incoming_unique_clients - x.inc_ucl_prev_week
    END AS wtw,
    CASE WHEN x.inc_ucl_prev_month IS NULL THEN NULL
         ELSE x.incoming_unique_clients - x.inc_ucl_prev_month
    END AS mtm
  FROM agg_with_lags x
  WHERE x.week_start = (SELECT MAX(week_start) FROM agg_with_lags)

  UNION ALL
  SELECT
    'SUMMARY',
    x.week_start,
    TO_CHAR(x.week_start, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
      || ' - '
      || TO_CHAR(x.week_start+6, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN'),
    NULL,
    NULL, NULL, NULL, NULL, NULL,
    'Уникальные клиенты (исх.)',
    x.outgoing_unique_clients,
    CASE WHEN x.out_ucl_prev_week IS NULL THEN NULL
         ELSE x.outgoing_unique_clients - x.out_ucl_prev_week
    END,
    CASE WHEN x.out_ucl_prev_month IS NULL THEN NULL
         ELSE x.outgoing_unique_clients - x.out_ucl_prev_month
    END
  FROM agg_with_lags x
  WHERE x.week_start = (SELECT MAX(week_start) FROM agg_with_lags)

  UNION ALL
  SELECT
    'SUMMARY',
    x.week_start,
    TO_CHAR(x.week_start, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
      || ' - '
      || TO_CHAR(x.week_start+6, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN'),
    NULL,
    NULL, NULL, NULL, NULL, NULL,
    'Средний перевод (вход.), млн. руб.',
    CASE
      WHEN x.incoming_unique_clients=0 THEN 0
      ELSE (x.incoming_sum_trans / x.incoming_unique_clients)/1e6
    END,
    CASE
      WHEN x.inc_ucl_prev_week IS NULL OR x.inc_ucl_prev_week=0 THEN NULL
      ELSE ((x.incoming_sum_trans / x.incoming_unique_clients)
           - (x.inc_sum_prev_week / x.inc_ucl_prev_week))/1e6
    END,
    CASE
      WHEN x.inc_ucl_prev_month IS NULL OR x.inc_ucl_prev_month=0 THEN NULL
      ELSE ((x.incoming_sum_trans / x.incoming_unique_clients)
           - (x.inc_sum_prev_month / x.inc_ucl_prev_month))/1e6
    END
  FROM agg_with_lags x
  WHERE x.week_start = (SELECT MAX(week_start) FROM agg_with_lags)

  UNION ALL
  SELECT
    'SUMMARY',
    x.week_start,
    TO_CHAR(x.week_start, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN')
      || ' - '
      || TO_CHAR(x.week_start+6, 'FMDD Mon', 'NLS_DATE_LANGUAGE=RUSSIAN'),
    NULL,
    NULL, NULL, NULL, NULL, NULL,
    'Средний перевод (исх.), млн. руб.',
    CASE
      WHEN x.outgoing_unique_clients=0 THEN 0
      ELSE (x.outgoing_sum_trans / x.outgoing_unique_clients)/1e6
    END,
    CASE
      WHEN x.out_ucl_prev_week IS NULL OR x.out_ucl_prev_week=0 THEN NULL
      ELSE ((x.outgoing_sum_trans / x.outgoing_unique_clients)
           - (x.out_sum_prev_week / x.out_ucl_prev_week))/1e6
    END,
    CASE
      WHEN x.out_ucl_prev_month IS NULL OR x.out_ucl_prev_month=0 THEN NULL
      ELSE ((x.outgoing_sum_trans / x.outgoing_unique_clients)
           - (x.out_sum_prev_month / x.out_ucl_prev_month))/1e6
    END
  FROM agg_with_lags x
  WHERE x.week_start = (SELECT MAX(week_start) FROM agg_with_lags)
)

-------------------------------------------------------------------------------
-- И финальный SELECT, где мы можем удобно заказывать порядок вывода
-------------------------------------------------------------------------------
SELECT
  block_type,
  week_start,
  week,
  bank_name,
  incoming_sum_trans,
  incoming_unique_clients,
  outgoing_sum_trans,
  outgoing_unique_clients,
  both_in_out_clients,
  measure,
  value,
  wtw,
  mtm
FROM union_result
ORDER BY
  CASE block_type WHEN 'DETAIL' THEN 1 WHEN 'SUMMARY' THEN 2 END,
  week_start,
  -- Чтобы внутри DETAIL сначала шли банки построчно, а "ALL BANKS" например последней:
  CASE WHEN bank_name='ALL BANKS' THEN 2 ELSE 1 END,
  measure
;
