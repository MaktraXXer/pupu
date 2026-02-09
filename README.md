DECLARE @dt_rep date = '2026-01-01';

/* список вкладов маркетплейсов */
DECLARE @mp TABLE (prod_name nvarchar(200) NOT NULL PRIMARY KEY);
INSERT INTO @mp(prod_name) VALUES
 (N'Надёжный прайм')
,(N'Надёжный VIP')
,(N'Надёжный премиум')
,(N'Надёжный промо')
,(N'Надёжный старт')
,(N'Надёжный Т2')
,(N'Надёжный Мегафон')
,(N'Могучий’')
,(N'Надёжный’')
,(N'Всё в ДОМ’')
,(N'ДОМа Надёжно’');

WITH base AS (
    SELECT
          t.con_id
        , t.SECTION_NAME
        , t.PROD_NAME_RES
        , t.OUT_RUB
        , t.rate_con
        , t.rate_trf
        , CAST(t.dt_open AS date)  AS dt_open
        , t.termdays
        , ISNULL(t.is_floatrate,0) AS is_floatrate
    FROM [ALM].[VW_balance_rest_all] t WITH (NOLOCK)
    WHERE 1=1
      AND t.dt_rep = @dt_rep
      AND t.OUT_RUB IS NOT NULL
      AND t.od_flag = 1
      AND t.block_name = 'Привлечение ФЛ'
      AND t.cur = '810'
),
enriched AS (
    SELECT
          b.*
        , CASE WHEN EXISTS (SELECT 1 FROM @mp m WHERE m.prod_name = b.PROD_NAME_RES) THEN 1 ELSE 0 END AS is_mp
        , LTRIM(RTRIM(b.SECTION_NAME)) AS section_name_norm
    FROM base b
),
labeled AS (
    SELECT
          e.*
        , CASE
            WHEN e.section_name_norm IN (N'Накопительный счёт', N'До востребования') THEN 1
            WHEN e.section_name_norm <> N'Срочные' THEN 99
            WHEN e.is_mp = 1 THEN 7
            WHEN e.dt_open BETWEEN '2025-10-01' AND '2025-11-05'
             AND e.rate_con BETWEEN 0.163 AND 0.17
             AND e.termdays BETWEEN 85 AND 110
             AND e.is_floatrate = 0
            THEN 2
            WHEN e.dt_open BETWEEN '2025-11-06' AND '2025-12-31'
             AND e.rate_con BETWEEN 0.163 AND 0.17
             AND e.termdays BETWEEN 55 AND 110
             AND e.is_floatrate = 0
            THEN 3
            WHEN e.dt_open >= '2026-01-01'
             AND e.rate_con BETWEEN 0.158 AND 0.16
             AND e.termdays BETWEEN 55 AND 110
             AND e.is_floatrate = 0
            THEN 5
            WHEN e.dt_open <= '2025-12-31' THEN 4
            WHEN e.dt_open >= '2026-01-01' THEN 6
            ELSE 99
          END AS bucket_id
    FROM enriched e
)
SELECT
      con_id
    , PROD_NAME_RES
    , SECTION_NAME
    , rate_con
    , rate_trf
    , OUT_RUB
    , dt_open
    , termdays
    , is_floatrate
FROM labeled
WHERE bucket_id = 99
ORDER BY OUT_RUB DESC;
