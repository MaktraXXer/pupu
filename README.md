DECLARE @dt_from_rep date = '2026-02-05';
DECLARE @dt_to_rep   date = '2026-03-14';

/* список вкладов маркетплейсов */
DECLARE @mp TABLE (
    prod_name nvarchar(200) NOT NULL PRIMARY KEY
);

INSERT INTO @mp(prod_name)
VALUES
    (N'Надёжный прайм'),
    (N'Надёжный VIP'),
    (N'Надёжный премиум'),
    (N'Надёжный промо'),
    (N'Надёжный старт'),
    (N'Надёжный Т2'),
    (N'Надёжный Мегафон'),
    (N'Могучий'),
    (N'Надёжный'),
    (N'Всё в ДОМ'),
    (N'ДОМа Надёжно');

WITH base AS (
    SELECT
          t.dt_rep
        , LTRIM(RTRIM(t.SECTION_NAME)) AS SECTION_NAME
        , t.OUT_RUB
        , CAST(t.dt_open AS date)      AS dt_open
        , t.rate_con
        , t.termdays
        , ISNULL(t.is_floatrate, 0)    AS is_floatrate
        , t.PROD_NAME_RES              AS PROD_NAME
    FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
    WHERE 1 = 1
      AND t.dt_rep >= @dt_from_rep
      AND t.dt_rep <= @dt_to_rep
      AND t.OUT_RUB IS NOT NULL
      AND t.od_flag = 1
      AND t.block_name = N'Привлечение ФЛ'
      AND t.cur = '810'
      AND LTRIM(RTRIM(t.SECTION_NAME)) IN (N'Накопительный счёт', N'До востребования', N'Срочные')
),
enriched AS (
    SELECT
          b.*
        , CASE
            WHEN EXISTS (
                SELECT 1
                FROM @mp m
                WHERE m.prod_name = b.PROD_NAME
            ) THEN 1
            ELSE 0
          END AS is_mp
    FROM base b
),
labeled AS (
    SELECT
          e.dt_rep
        , e.OUT_RUB
        , CASE
            /* НС + ДВС вместе */
            WHEN e.SECTION_NAME IN (N'Накопительный счёт', N'До востребования') THEN 1

            /* дальше только Срочные могли остаться либо ошибка */
            WHEN e.SECTION_NAME <> N'Срочные' THEN 99

            /* выпишем маркетплейсы сразу, не интересна динамика новых маркетов */
            WHEN e.is_mp = 1 THEN 7

            /* 2 категория - 01.10–05.11.2025, 0.163–0.17, 85–110, fixed, НЕ маркетплейсы */
            WHEN e.dt_open BETWEEN '2025-10-01' AND '2025-11-05'
             AND e.rate_con BETWEEN 0.163 AND 0.17
             AND e.termdays BETWEEN 85 AND 110
             AND e.is_floatrate = 0
            THEN 2

            /* 3 категория - 06.11–31.12.2025, 0.163–0.17, 55–110, fixed, НЕ маркетплейсы */
            WHEN e.dt_open BETWEEN '2025-11-06' AND '2025-12-31'
             AND e.rate_con BETWEEN 0.163 AND 0.17
             AND e.termdays BETWEEN 55 AND 110
             AND e.is_floatrate = 0
            THEN 3

            /* 4 - остальные до 31.12.2025, НЕ маркетплейсы */
            WHEN e.dt_open <= '2025-12-31' THEN 4

            /* 8 - РК МАРТ: та же фильтрация, что и пункт 5, но открыты с 01.03.2026 */
            WHEN e.dt_open >= '2026-03-01'
             AND e.rate_con BETWEEN 0.158 AND 0.16
             AND e.termdays BETWEEN 55 AND 110
             AND e.is_floatrate = 0
            THEN 8

            /* 5 - с 01.01.2026 по 29.02.2026 fixed 0.158–0.16 55–110, НЕ маркетплейсы */
            WHEN e.dt_open BETWEEN '2026-01-01' AND '2026-02-28'
             AND e.rate_con BETWEEN 0.158 AND 0.16
             AND e.termdays BETWEEN 55 AND 110
             AND e.is_floatrate = 0
            THEN 5

            /* 6 - с 01.01.2026, НЕ маркетплейсы, но не попало в 5 и 8 */
            WHEN e.dt_open >= '2026-01-01' THEN 6

            ELSE 99
          END AS bucket_id
    FROM enriched e
)
SELECT
      dt_rep
    , bucket_id
    , CASE bucket_id
        WHEN 1 THEN N'1) НС + ДВС'
        WHEN 2 THEN N'2) 01.10–05.11.2025 fixed 0.163–0.17 85–110 НЕ маркетплейсы'
        WHEN 3 THEN N'3) 06.11–31.12.2025 fixed 0.163–0.17 55–110 НЕ маркетплейсы'
        WHEN 4 THEN N'4) До 31.12.2025 прочие НЕ маркетплейсы'
        WHEN 5 THEN N'5) 01.01–29.02.2026 fixed 0.158–0.16 55–110 НЕ маркетплейсы'
        WHEN 6 THEN N'6) С 01.01.2026 прочие НЕ маркетплейсы'
        WHEN 7 THEN N'7) Маркетплейсы (список)'
        WHEN 8 THEN N'8) РК МАРТ: с 01.03.2026 fixed 0.158–0.16 55–110 НЕ маркетплейсы'
        ELSE N'99) НЕ РАЗОБРАНО'
      END AS bucket_name
    , SUM(OUT_RUB) / 1e9 AS out_rub_bln
FROM labeled
GROUP BY dt_rep, bucket_id
ORDER BY dt_rep, bucket_id;
