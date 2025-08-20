WITH map_products AS (
    /* code = TRUNC(rt.amountfrom) из источника */
    SELECT 51 AS code, 'Надёжный'          AS prod_name,  224  AS prod_id FROM dual UNION ALL
    SELECT 52        , 'Надёжный Промо'               , 3083              FROM dual UNION ALL
    SELECT 53        , 'ДОМа надёжно'                 , 3082              FROM dual UNION ALL
    SELECT 54        , 'Надёжный старт'               , 3155              FROM dual UNION ALL
    SELECT 55        , 'Надёжный премиум'             , 3081              FROM dual UNION ALL
    SELECT 56        , 'Всё в ДОМ'                    , 3156              FROM dual UNION ALL
    SELECT 57        , 'Надёжный VIP'                 , 3080              FROM dual UNION ALL
    SELECT 60        , 'Надёжный Т2'                  , 3182              FROM dual UNION ALL
    SELECT 91        , 'Надёжный Мегафон'             , 3186              FROM dual UNION ALL
    SELECT 92        , 'Надёжный прайм'               , 3195              FROM dual UNION ALL
    SELECT 93        , 'Надёжный процент'             , 3194              FROM dual
)
SELECT
    rv.ValidFromDate         AS DT_FROM,
    rv.ValidToDate - 1       AS DT_TO,
    TRUNC(rd.amountfrom)     AS TERMDAYS_FROM,
    TRUNC(rd.amountto) - 1   AS TERMDAYS_TO,
    mp.prod_id               AS PROD_ID,
    mp.prod_name             AS PROD_NAME,
    rt.percrate              AS PERCRATE,
    SYSTIMESTAMP             AS load_dt
FROM ods_001.ComputeRate     crt
JOIN ods_001.CRateVersion    rv ON rv.Rate        = crt.Classified
JOIN ods_001.CRateDependence rd ON rd.RateVersion = rv.Classified
JOIN ods_001.CRateTable      rt ON rt.RateVariant = rd.Classified
JOIN map_products            mp ON mp.code        = TRUNC(rt.amountfrom)
WHERE crt.label        = '% Клиентское вознаграждение (RUR)'
  AND crt.active_flag  = 'Y' AND crt.deleted_flag  = 'N'
  AND rv.active_flag   = 'Y' AND rv.deleted_flag   = 'N'
  AND rd.active_flag   = 'Y' AND rd.deleted_flag   = 'N'
  AND rt.active_flag   = 'Y' AND rt.deleted_flag   = 'N'
  -- исходные закомментированные условия оставлены как есть:
  -- AND DATE '2025-07-31' BETWEEN rv.ValidFromDate AND rv.ValidToDate - INTERVAL '0.01' DAY
  -- AND nDpnd BETWEEN rd.AmountFrom AND rd.AmountTo - 1
  -- AND (nAmtFrom IS NULL OR nAmtFrom = rt.AmountFrom)
ORDER BY mp.prod_name, DT_FROM, TERMDAYS_FROM;
