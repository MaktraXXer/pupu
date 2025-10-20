/* Итог "tall": 
   — строка 91 = ТОЛЬКО не-РК (is_91_rk = 0)
   — отдельная строка "91 РК" = только РК (is_91_rk = 1)
   — wavg по rate_con/rate_trf только где ставка NOT NULL (через wsum/wden) */
tall AS (
    /* Все сроки, КРОМЕ РК-части для 91 */
    SELECT
        CAST(term_bucket AS nvarchar(20)) AS [Срок, дн.],
        dt_open_d                         AS [Дата открытия],
        SUM(out_rub)                      AS [Объем, руб.],
        CAST(SUM(wsum_rate_con) / NULLIF(SUM(wden_rate_con),0) AS decimal(9,6)) AS [Средневзв. ставка (клиент)],
        CAST(SUM(wsum_rate_trf) / NULLIF(SUM(wden_rate_trf),0) AS decimal(9,6)) AS [Средневзв. ставка (ТС)]
    FROM flag_91rk
    WHERE term_bucket <> 91                      -- любые сроки, кроме 91
       OR (term_bucket = 91 AND is_91_rk = 0)    -- срок 91, НО только не-РК
    GROUP BY term_bucket, dt_open_d

    UNION ALL

    /* Спец-строка: только 91 РК */
    SELECT
        N'91 РК'                          AS [Срок, дн.],
        f.dt_open_d                       AS [Дата открытия],
        SUM(f.out_rub)                    AS [Объем, руб.],
        CAST(SUM(f.wsum_rate_con) / NULLIF(SUM(f.wden_rate_con),0) AS decimal(9,6)) AS [Средневзв. ставка (клиент)],
        CAST(SUM(f.wsum_rate_trf) / NULLIF(SUM(f.wden_rate_trf),0) AS decimal(9,6)) AS [Средневзв. ставка (ТС)]
    FROM flag_91rk f
    WHERE f.term_bucket = 91 AND f.is_91_rk = 1
    GROUP BY f.dt_open_d
)
SELECT *
FROM tall
ORDER BY [Дата открытия],
         CASE WHEN [Срок, дн.] = N'91 РК' THEN 2147483647 ELSE TRY_CONVERT(int, [Срок, дн.]) END,
         [Срок, дн.];
