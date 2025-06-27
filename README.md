/* ==== Показатели ставок + итоговая сумма по периоду dt_from-dt_to ===== */

SELECT
       t.dt_from,
       t.dt_to,

       /*--- отдельные компоненты ---*/
       TRR_BASE_RATE           = MAX(CASE WHEN t.TRF_RATE_TYPE = 'BASE_RATE'           THEN t.TRF_RATE END),
       TRR_LIQUIDITY_RATE      = MAX(CASE WHEN t.TRF_RATE_TYPE = 'LIQUIDITY_RATE'      THEN t.TRF_RATE END),
       TRR_OPTION_RATE         = MAX(CASE WHEN t.TRF_RATE_TYPE = 'OPTION_RATE'         THEN t.TRF_RATE END),
       TRR_LIQUIDITY_PREF_RATE = MAX(CASE WHEN t.TRF_RATE_TYPE = 'LIQUIDITY_PREF_RATE' THEN t.TRF_RATE END),
       TRR_SUBSID_RATE         = MAX(CASE WHEN t.TRF_RATE_TYPE = 'SUBSID_RATE'         THEN t.TRF_RATE END),
       TRR_FOR_RATE            = MAX(CASE WHEN t.TRF_RATE_TYPE = 'FOR_RATE'            THEN t.TRF_RATE END),
       TRR_SSV_RATE            = MAX(CASE WHEN t.TRF_RATE_TYPE = 'SSV_RATE'            THEN t.TRF_RATE END),

       /*--- итоговая ставка = сумма всех компонент ---*/
       SUM_RATE =  ISNULL(MAX(CASE WHEN t.TRF_RATE_TYPE = 'BASE_RATE'           THEN t.TRF_RATE END),0)
                + ISNULL(MAX(CASE WHEN t.TRF_RATE_TYPE = 'LIQUIDITY_RATE'      THEN t.TRF_RATE END),0)
                + ISNULL(MAX(CASE WHEN t.TRF_RATE_TYPE = 'OPTION_RATE'         THEN t.TRF_RATE END),0)
                + ISNULL(MAX(CASE WHEN t.TRF_RATE_TYPE = 'LIQUIDITY_PREF_RATE' THEN t.TRF_RATE END),0)
                + ISNULL(MAX(CASE WHEN t.TRF_RATE_TYPE = 'SUBSID_RATE'         THEN t.TRF_RATE END),0)
                + ISNULL(MAX(CASE WHEN t.TRF_RATE_TYPE = 'FOR_RATE'            THEN t.TRF_RATE END),0)
                + ISNULL(MAX(CASE WHEN t.TRF_RATE_TYPE = 'SSV_RATE'            THEN t.TRF_RATE END),0)
FROM   dm_trf.trf_con_rate_demand t WITH (NOLOCK)
WHERE  t.con_id = '20183830'                -- нужный договор
  AND  t.TRF_RATE_TYPE IN ( 'BASE_RATE',
                            'LIQUIDITY_RATE',
                            'OPTION_RATE',
                            'LIQUIDITY_PREF_RATE',
                            'SUBSID_RATE',
                            'FOR_RATE',
                            'SSV_RATE')
GROUP BY t.dt_from, t.dt_to
ORDER BY t.dt_from, t.dt_to;
