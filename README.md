BEGIN TRANSACTION;

-- === 1. Обновление значений rate_trf_with_nadbavka ===
UPDATE tgt
SET tgt.value = ctrl.value + ndb.value
FROM [ALM_TEST].[alm_history].[interest_rates] AS tgt
INNER JOIN [ALM_TEST].[alm_history].[interest_rates] AS ctrl
    ON  ctrl.dt_from   = tgt.dt_from
    AND ctrl.dt_to     = tgt.dt_to
    AND ctrl.cur       = tgt.cur
    AND ctrl.term      = tgt.term
    AND ctrl.conv      = tgt.conv
    AND ctrl.rate_type = 'rate_trf_controlling'
INNER JOIN [ALM_TEST].[alm_history].[interest_rates] AS ndb
    ON  ndb.dt_from    = tgt.dt_from
    AND ndb.dt_to      = tgt.dt_to
    AND ndb.cur        = tgt.cur
    AND ndb.term       = tgt.term
    AND ndb.conv       = tgt.conv
    AND ndb.rate_type  = 'nadbavka'
WHERE tgt.rate_type = 'rate_trf_with_nadbavka'
  AND tgt.cur = 810
  AND tgt.conv = 'AT_THE_END'
  AND tgt.dt_from = '2025-09-15';

-- === 2. Проверка результата ===
SELECT dt_from, term, cur, conv, rate_type, value
FROM [ALM_TEST].[alm_history].[interest_rates]
WHERE dt_from = '2025-09-15'
  AND cur = 810
  AND conv = 'AT_THE_END'
  AND rate_type IN ('rate_trf_controlling', 'nadbavka', 'rate_trf_with_nadbavka')
ORDER BY term, rate_type;

--COMMIT TRANSACTION;  -- раскомментировать после проверки
--ROLLBACK TRANSACTION; -- если нужно отменить
