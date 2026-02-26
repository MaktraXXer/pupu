DECLARE @DateFrom date = '2026-02-01';
DECLARE @DateTo   date = CONVERT(date, GETDATE()-2);

;WITH prod_ref AS (
    SELECT 'Надёжный прайм' AS prod_name UNION ALL
    SELECT 'Надёжный VIP' UNION ALL
    SELECT 'Надёжный премиум' UNION ALL
    SELECT 'Надёжный промо' UNION ALL
    SELECT 'Надёжный старт' UNION ALL
    SELECT 'Надёжный Т2' UNION ALL
    SELECT 'Надёжный Мегафон' UNION ALL
    SELECT 'Могучий' UNION ALL
    SELECT 'ДОМа надёжно' UNION ALL
    SELECT 'Всё в ДОМ'
)
SELECT
    t.dt_rep,
    SUM(t.out_rub) AS out_rub_total,

    -- твоя средневзвешенная ETS-ставка (как было)
    SUM(
        CASE
            WHEN t.rate_trf IS NOT NULL AND t.rate_con IS NOT NULL THEN
                (
                    CASE
                        WHEN t.rate_trf >= t.rate_con
                            THEN (t.rate_trf + 0.0048)/0.9525
                        ELSE
                            CASE
                                WHEN pr.prod_name IS NULL
                                    THEN (t.rate_con + 0.0010 + 0.0048)/0.9525
                                ELSE (t.rate_trf + 0.0048)/0.9525
                            END
                    END
                ) * t.out_rub
            ELSE 0
        END
    )
    / NULLIF(
        SUM(CASE WHEN t.rate_trf IS NOT NULL AND t.rate_con IS NOT NULL THEN t.out_rub END),
        0
      ) AS rate_ets_wavg,

    -- НОВОЕ: средневзвешенный AVG_KEY_RATE по объёму
    SUM(
        CASE
            WHEN c.avg_key_rate IS NOT NULL THEN c.avg_key_rate * t.out_rub
            ELSE 0
        END
    )
    / NULLIF(
        SUM(CASE WHEN c.avg_key_rate IS NOT NULL THEN t.out_rub END),
        0
      ) AS avg_key_rate_wavg,

    -- (опционально) контроль качества матчинга
    SUM(CASE WHEN t.termdays IS NULL THEN t.out_rub ELSE 0 END) AS out_rub_term_null,
    SUM(CASE WHEN t.termdays IS NOT NULL AND c.avg_key_rate IS NULL THEN t.out_rub ELSE 0 END) AS out_rub_no_cache_match

FROM alm.[ALM].[vw_balance_rest_all] AS t WITH (NOLOCK)
LEFT JOIN prod_ref pr
       ON pr.prod_name = t.prod_name_res
LEFT JOIN WORK.ForecastKey_Cache c WITH (NOLOCK)
       ON c.dt_rep = CONVERT(date, t.dt_open)   -- дата открытия = dt_rep прогноза
      AND c.term  = t.termdays                  -- срочность

WHERE
    t.dt_rep BETWEEN @DateFrom AND @DateTo
    AND CONVERT(date, t.dt_open) = t.dt_rep     -- только «новые в этот же день»
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag      = 1
    AND t.is_floatrate = 0
    AND t.cur          = '810'
    AND t.out_rub IS NOT NULL

GROUP BY t.dt_rep
ORDER BY t.dt_rep
OPTION (RECOMPILE);
