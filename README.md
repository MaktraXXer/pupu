да, переписал так, чтобы на выходе был ровно ваш целевой формат (DT_FROM, DT_TO, TERMDAYS_FROM, TERMDAYS_TO, PROD_ID, PROD_NAME, PERCRATE, load_dt) и при этом брались только нужные продукты через маппинг «label → PROD_NAME/PROD_ID».

ORACLE SQL (одним запросом)

-- === Маппинг: label из источника → целевой PROD_NAME и PROD_ID ===
WITH map AS (
    SELECT 'Надёжный прайм'  AS label, 'Надёжный прайм'  AS prod_name, 3195 AS prod_id FROM dual UNION ALL
    SELECT 'Надёжный процент',          'Надёжный процент',          3194 FROM dual UNION ALL
    SELECT 'Надёжный Промо',            'Надёжный Промо',            3083 FROM dual UNION ALL
    SELECT 'Надёжный премиум',          'Надёжный премиум',          3081 FROM dual UNION ALL
    SELECT 'ДОМа надёжно',              'ДОМа надёжно',              3082 FROM dual UNION ALL
    -- В источнике встречается "Надёжный t2" → приводим к "Надёжный Т2"
    SELECT 'Надёжный t2',               'Надёжный Т2',               3182 FROM dual UNION ALL
    SELECT 'Надёжный Мегафон',          'Надёжный Мегафон',          3186 FROM dual UNION ALL
    SELECT 'Надёжный старт',            'Надёжный старт',            3155 FROM dual UNION ALL
    SELECT 'Надёжный VIP',              'Надёжный VIP',              3080 FROM dual UNION ALL
    -- В источнике без точки над "ё": "Все в ДОМ" → нормализуем к "Всё в ДОМ"
    SELECT 'Все в ДОМ',                 'Всё в ДОМ',                 3156 FROM dual UNION ALL
    SELECT 'Надёжный',                  'Надёжный',                   224 FROM dual
),
-- Вспомогательная таблица код→label из вашего примера
prod AS (
    SELECT '51' AS code, 'Надёжный'          AS label FROM dual UNION ALL
    SELECT '52',          'Надёжный Промо'                FROM dual UNION ALL
    SELECT '53',          'ДОМа надёжно'                  FROM dual UNION ALL
    SELECT '54',          'Надёжный старт'                FROM dual UNION ALL
    SELECT '55',          'Надёжный премиум'              FROM dual UNION ALL
    SELECT '56',          'Все в ДОМ'                     FROM dual UNION ALL
    SELECT '57',          'Надёжный VIP'                  FROM dual UNION ALL
    SELECT '60',          'Надёжный t2'                   FROM dual UNION ALL
    SELECT '61',          'ПК sravni'                     FROM dual UNION ALL
    SELECT '62',          'ПК banki'                      FROM dual UNION ALL
    SELECT '63',          'ПК vbr'                        FROM dual UNION ALL
    SELECT '91',          'Надёжный Мегафон'              FROM dual UNION ALL
    SELECT '92',          'Надёжный прайм'                FROM dual UNION ALL
    SELECT '93',          'Надёжный процент'              FROM dual
)
SELECT
    -- даты действия
    rv.ValidFromDate                          AS DT_FROM,
    rv.ValidToDate - 1                        AS DT_TO,
    -- границы сроков (нижняя включ., верхняя экскл. → минус 1)
    TRUNC(rd.amountfrom)                      AS TERMDAYS_FROM,
    TRUNC(rd.amountto) - 1                    AS TERMDAYS_TO,
    -- итоговые идентификаторы продукта из маппинга
    m.prod_id                                 AS PROD_ID,
    m.prod_name                               AS PROD_NAME,
    -- искомая надбавка
    rt.percrate                               AS PERCRATE,
    SYSTIMESTAMP                              AS load_dt
FROM ods_001.ComputeRate     crt
JOIN ods_001.CRateVersion    rv ON rv.Rate        = crt.Classified
JOIN ods_001.CRateDependence rd ON rd.RateVersion = rv.Classified
JOIN ods_001.CRateTable      rt ON rt.RateVariant = rd.Classified
JOIN prod                         ON prod.code    = TRUNC(rt.amountfrom)
JOIN map  m                       ON m.label      = prod.label   -- отбираем ТОЛЬКО нужные продукты
WHERE crt.label        = '% Клиентское вознаграждение (RUR)'
  AND crt.active_flag  = 'Y' AND crt.deleted_flag  = 'N'
  AND rv.active_flag   = 'Y' AND rv.deleted_flag   = 'N'
  AND rd.active_flag   = 'Y' AND rd.deleted_flag   = 'N'
  AND rt.active_flag   = 'Y' AND rt.deleted_flag   = 'N'
-- при необходимости можно вернуть фильтр по дате действия:
--  AND DATE '2025-07-31' BETWEEN rv.ValidFromDate AND rv.ValidToDate - INTERVAL '0.01' DAY
ORDER BY m.prod_name, DT_FROM, TERMDAYS_FROM;

	•	Мы не берём «ПК …» продукты, потому что у них нет соответствий в map (inner join отфильтрует).
	•	Все «тонкости» из примера сохранены: DT_TO = ValidToDate - 1, TERMDAYS_TO = amountto - 1.

⸻

(опционально) Загрузка в SQL Server ALM_TEST.markets.prod_term_rates

Если тянете через Linked Server, можно сразу заливать:

-- В SQL Server
INSERT INTO [ALM_TEST].[markets].[prod_term_rates] (
    PROD_NAME, PROD_ID, TERMDAYS_FROM, TERMDAYS_TO, DT_FROM, DT_TO, PERCRATE, load_dt
)
SELECT PROD_NAME, PROD_ID, TERMDAYS_FROM, TERMDAYS_TO, DT_FROM, DT_TO, PERCRATE, load_dt
FROM OPENQUERY(ORCL, '
    -- <<вставьте сюда Oracle SELECT из блока выше>>
');

Нужно иначе (например, у вас уже есть «сырой» дамп из Oracle в staging-таблице) — скажи, дам точный T-SQL MERGE под вашу staging-структуру.
