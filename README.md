WITH bic_map AS (
  SELECT
    cb.bik,
    c.cli_short_name,
    ROW_NUMBER() OVER (PARTITION BY cb.bik ORDER BY cb.cli_id) AS rn
  FROM dds.client_bank cb
  JOIN dds.client c
    ON c.cli_id = cb.cli_id
)
SELECT
  doc.doc_id,
  doc.dt_receipt,
  e.vl_rub,
  doc.cli_id_payer,
  --c.cli_name,
  doc.payer_name,
  doc.recip_name,
  doc.payer_bank_bic,
  doc.recip_bank_bic,
  bm.cli_short_name,
  -- Приведение наименования банка согласно той же логике
  CASE
    WHEN UPPER(bm.cli_short_name) LIKE '%СБЕРБАНК%' THEN 'Сбербанк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ВБРР%' THEN 'Банк ВБРР'
    WHEN UPPER(bm.cli_short_name) LIKE '%ВТБ%'      THEN 'ВТБ'
    WHEN UPPER(bm.cli_short_name) LIKE '%АЛЬФА%-БАНК%' OR
         UPPER(bm.cli_short_name) LIKE '%АЛЬФА БАНК%' THEN 'Альфа Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%АК БАРС%'    THEN 'Ак Барс Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%УРАЛСИБ%'    THEN 'БАНК УРАЛСИБ'
    WHEN UPPER(bm.cli_short_name) LIKE '%ГАЗПРОМБАНК%' OR
         UPPER(bm.cli_short_name) LIKE '%ГПБ%'         THEN 'Газпромбанк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ПРОМСВЯЗЬ%'   THEN 'Промсвязьбанк'
    WHEN UPPER(bm.cli_short_name) LIKE '%РОСБАНК%'     THEN 'Росбанк'
    WHEN UPPER(bm.cli_short_name) LIKE '%РНКБ%'        THEN 'РНКБ Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%РАЙФФАЙЗЕН%'  THEN 'Райффайзен Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%РОССЕЛЬХОЗ%'   THEN 'Россельхозбанк'
    WHEN UPPER(bm.cli_short_name) LIKE '%СОВКОМБАНК%'   THEN 'Совкомбанк'
    WHEN UPPER(bm.cli_short_name) LIKE '%МОСКОВСКИЙ КРЕДИТНЫЙ БАНК%' OR
         UPPER(bm.cli_short_name) LIKE '%МКБ%'         THEN 'МКБ (Московский кредитный банк)'
    WHEN UPPER(bm.cli_short_name) LIKE '%ЮНИКРЕДИТ%'    THEN 'ЮниКредит Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ЭКСПОБАНК%'     THEN 'Экспобанк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ОТП БАНК%'       THEN 'ОТП Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ОЗОН БАНК%'       THEN 'Озон Банк (Ozon)'
    WHEN UPPER(bm.cli_short_name) LIKE '%ПОЧТА БАНК%'      THEN 'Почта Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%НБД%-БАНК%'       THEN 'НБД-Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%СДМ%-БАНК%'       THEN 'СДМ-Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ЛОКО%-БАНК%'       THEN 'КБ ЛОКО-Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%УБРИР%'          THEN 'УБРиР'
    WHEN UPPER(bm.cli_short_name) LIKE '%УРАЛ ФД%'        THEN 'Урал ФД'
    WHEN UPPER(bm.cli_short_name) LIKE '%УРАЛПРОМБАНК%'   THEN 'АО "УРАЛПРОМБАНК"'
    WHEN UPPER(bm.cli_short_name) LIKE '%АБСОЛЮТ%'        THEN 'Абсолют Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%РУССКИЙ СТАНДАРТ%' THEN 'Банк Русский Стандарт'
    WHEN UPPER(bm.cli_short_name) LIKE '%БАНК СИНАРА%'     THEN 'Банк Синара'
    WHEN UPPER(bm.cli_short_name) LIKE '%МТС%-БАНК%'       THEN 'МТС-Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%БАНК ЗЕНИТ%'      THEN 'Банк ЗЕНИТ'
    WHEN UPPER(bm.cli_short_name) LIKE '%МОДУЛЬБАНК%'      THEN 'КБ Модульбанк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ФК ОТКРЫТИЕ%'     THEN 'Банк ФК Открытие'
    WHEN UPPER(bm.cli_short_name) LIKE '%КРЕДИТ ЕВРОПА%'   THEN 'Кредит Европа Банк (Россия)'
    WHEN UPPER(bm.cli_short_name) LIKE '%АЗИАТСКО%-ТИХООКЕАНСКИЙ%' THEN 'Азиатско-Тихоокеанский Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ББР БАНК%'        THEN 'ББР Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ТАВРИЧЕСКИЙ%'     THEN 'Таврический Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ИНГОССТРАХ%'      THEN 'Ингосстрах Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ХОУМ КРЕДИТ%'      THEN 'Хоум Кредит Банк (Хоум Банк)'
    WHEN UPPER(bm.cli_short_name) LIKE '%КУБАНЬ КРЕДИТ%'     THEN 'Кубань Кредит'
    WHEN UPPER(bm.cli_short_name) LIKE '%МЕТАЛЛИНВЕСТБАНК%'  THEN 'Металлинвестбанк'
    WHEN UPPER(bm.cli_short_name) LIKE '%НОВИКОМБАНК%'       THEN 'НОВИКОМБАНК'
    WHEN UPPER(bm.cli_short_name) LIKE '%ТИНЬКОФФ%'         THEN 'Тинькофф Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%РОСТФИНАНС%'       THEN 'КБ  РостФинанс'
    WHEN UPPER(bm.cli_short_name) LIKE '%СОЛИДАРНОСТ%'       THEN 'КБ Солидарность'
    WHEN UPPER(bm.cli_short_name) LIKE '%ЦИФРА БАНК%'        THEN 'ФФИН Банк (Цифра банк)'
    WHEN UPPER(bm.cli_short_name) LIKE '%ТАТСОЦБАНК%'        THEN 'ТАТСОЦБАНК'
    WHEN UPPER(bm.cli_short_name) LIKE '%ТРАНСКАПИТАЛБАНК%'   THEN 'ТРАНСКАПИТАЛБАНК'
    WHEN UPPER(bm.cli_short_name) LIKE '%ТОЧКА%'            THEN 'Точка Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%СВОЙ БАНК%'          THEN 'Свой Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%СИТИБАНК%'           THEN 'Ситибанк'
    WHEN UPPER(bm.cli_short_name) LIKE '%РЕНЕССАНС%'          THEN 'Ренессанс Кредит'
    WHEN UPPER(bm.cli_short_name) LIKE '%ДАЛЬНЕВОСТОЧНЫЙ БАНК%' THEN 'Дальневосточный банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%ЗОЛОТАЯ КОРОНА%'      THEN 'Золотая Корона - KoronaPay (РНКО Платежный Центр)'
    WHEN UPPER(bm.cli_short_name) LIKE '%ЯНДЕКС БАНК%'         THEN 'Яндекс Банк'
    WHEN UPPER(bm.cli_short_name) LIKE '%АБ "РОССИЯ"%'         THEN 'АБ РОССИЯ'
    ELSE bm.cli_short_name
  END AS bank_name
FROM dds.doc_order doc
JOIN dds.entry e
  ON doc.doc_id = e.doc_id
JOIN dds.client c
  ON doc.cli_id_payer = c.cli_id
LEFT JOIN bic_map bm
  ON bm.bik = doc.recip_bank_bic
  AND bm.rn = 1
WHERE e.cur_db = '810'
  AND e.cur_cr = '810'
  AND doc.doc_trans_direct = 'OUTGOING'
  AND c.cli_type = 'I'
  -- Исключаем переводы, где в наименованиях плательщика/получателя встречается ИП
  AND NOT REGEXP_LIKE(UPPER(doc.payer_name), '(^|\\s)(ИП|IP)(\\s|$)')
  AND NOT REGEXP_LIKE(UPPER(doc.recip_name), '(^|\\s)(ИП|IP)(\\s|$)')
  -- Фильтр для сличения клиентов по нескольким параметрам
  AND (doc.cli_id_payer = doc.cli_id_recip 
       OR doc.payer_inn = doc.recip_inn
       OR REGEXP_REPLACE(UPPER(doc.payer_name), '[^A-ZА-Я]', '') =
          REGEXP_REPLACE(UPPER(doc.recip_name), '[^A-ZА-Я]', ''))
  AND doc.payer_bank_bic <> doc.recip_bank_bic
  AND doc.dt_receipt != DATE '4444-01-01'
  -- Период: с 27 февраля по 5 марта (исключаем 6 марта)
  AND doc.dt_receipt >= TO_DATE('13.03.2025', 'DD.MM.YYYY')
  AND doc.dt_receipt <  TO_DATE('20.03.2025', 'DD.MM.YYYY')
  AND e.vl_rub >= 100000
  -- Фильтр для ОТП Банка по наименованию
  AND UPPER(bm.cli_short_name) LIKE '%ВТБ%'
ORDER BY doc.dt_receipt, doc.doc_id;
