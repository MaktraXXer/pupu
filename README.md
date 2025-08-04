WITH base AS (
    SELECT
        dt_open,
        rate_trf,
        rate_trf_src,
        out_rub
    FROM [ALM].[ALM].[balance_rest_all] WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-08-02'
        AND OUT_RUB IS NOT NULL
        AND od_flag = 1
        AND block_name = 'Привлечение ФЛ'
        AND section_name IN ('Срочные')
        AND dt_open BETWEEN CAST(GETDATE() - 6 AS DATE) AND CAST(GETDATE() - 2 AS DATE)
)
SELECT
    dt_open,
    rate_trf,
    rate_trf_src,
    COUNT(*) AS deal_count,
    SUM(out_rub) AS total_out_rub,
    SUM(out_rub) * 1.0 / SUM(SUM(out_rub)) OVER (PARTITION BY dt_open) AS share_per_dt_open
FROM base
GROUP BY dt_open, rate_trf, rate_trf_src
ORDER BY dt_open, share_per_dt_open DESC;
