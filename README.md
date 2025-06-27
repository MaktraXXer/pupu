/* ------ итог по всем строкам выбранного договора --------------- */
;WITH rates AS (
    SELECT
          t.dt_from
        , t.dt_to
        , ISNULL(t.TRR_BASE_RATE           ,0) +
          ISNULL(t.TRR_LIQUIDITY_RATE      ,0) +
          ISNULL(t.TRR_OPTION_RATE         ,0) +
          ISNULL(t.TRR_LIQUIDITY_PREF_RATE ,0) +
          ISNULL(t.TRR_SUBSID_RATE         ,0) +
          ISNULL(t.TRR_FOR_RATE            ,0) +
          ISNULL(t.TRR_SSV_RATE            ,0)       AS rate_part
    FROM   dm_trf.trf_con_rate_demand  t  WITH (NOLOCK)
    WHERE  t.con_id = '20183830'
       -- если нужны только конкретные типы, раскомментируйте строку ниже
       -- AND  t.TRF_RATE_TYPE IN ('BASE_RATE','FOR_RATE','SSV_RATE')
)
SELECT
       dt_from,
       dt_to,
       SUM(rate_part) AS total_rate        -- общий итог по периоду
FROM   rates
GROUP BY dt_from, dt_to
ORDER BY dt_from, dt_to;
