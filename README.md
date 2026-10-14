USE ALM;
SET NOCOUNT ON;

/* PARAMETERS */
DECLARE @dt_to_mar date = '2026-03-12';
DECLARE @dt_to_sep date = '2026-09-12';
DECLARE @dt_to_oct date = '2026-10-12';

DECLARE @threshold decimal(18,2) = 200000000;

;WITH periods AS (
    SELECT CAST('2026-03-01' AS date) AS dt_from,
           @dt_to_mar AS dt_to,
           '01.03-' + CONVERT(varchar(5), @dt_to_mar, 104) AS r_w

    UNION ALL

    SELECT '2026-09-01',
           @dt_to_sep,
           '01.09-' + CONVERT(varchar(5), @dt_to_sep, 104)

    UNION ALL

    SELECT '2026-10-01',
           @dt_to_oct,
           '01.10-' + CONVERT(varchar(5), @dt_to_oct, 104)
),

base AS (
    SELECT
          p.r_w
        , t.bank_name_main
        , t.spec_cat
        , t.transaction_type
        , t.direction_txt
        , t.[распознано_НС]
          + t.[распознано_СР]
          + t.[Распознано_ДВС] AS amt

    FROM ALM.ehd.VW_transfers_FL_AGG_tab t WITH (NOLOCK)

    INNER JOIN periods p
        ON t.dt_rep BETWEEN p.dt_from AND p.dt_to

    WHERE t.direction_type <> N'Ошибка'
      AND t.[целевая категория] = 1
      AND t.transaction_type <> N'Внутренние переводы'
      AND t.transit_flag = N'Не транзит'
      AND t.cli_biz IN (N'Розница', N'ДЧБО')
),

flows AS (
    SELECT
          r_w

        , CASE
            WHEN spec_cat IN ('BR','SR','FU')
                THEN N'Market'
            WHEN transaction_type = N'Наличные'
                THEN N'Наличные'
            ELSE bank_name_main
          END AS bank

        , CASE
            WHEN spec_cat IN ('BR','SR','FU') THEN 'market'
            WHEN transaction_type = N'Наличные' THEN 'cash'
            ELSE 'bank'
          END AS bank_type

        , CASE
            WHEN direction_txt = N'входящие'
              OR (direction_txt = N'смешанные' AND amt > 0)
                THEN amt
            ELSE 0
          END AS incoming

        , CASE
            WHEN direction_txt = N'исходящие'
              OR (direction_txt = N'смешанные' AND amt < 0)
                THEN amt
            ELSE 0
          END AS outgoing

    FROM base

    WHERE spec_cat NOT IN ('BR','SR','FU')
       OR (
            spec_cat IN ('BR','SR','FU')
            AND transaction_type <> N'Наличные'
          )
),

by_bank AS (
    SELECT
          r_w
        , bank
        , bank_type
        , SUM(incoming) AS incoming
        , SUM(outgoing) AS outgoing

    FROM flows
    GROUP BY r_w, bank, bank_type
),

classified AS (
    SELECT
          r_w

        , CASE
            WHEN bank_type = 'bank'
             AND ABS(incoming) <= @threshold
             AND ABS(outgoing) <= @threshold
                THEN N'else'
            ELSE bank
          END AS bank_name_main

        , incoming
        , outgoing

    FROM by_bank
)

SELECT
      r_w
    , bank_name_main
    , SUM(incoming) / 1000000.0 AS incoming
    , SUM(outgoing) / 1000000.0 AS outgoing
    , SUM(incoming + outgoing) / 1000000.0 AS saldo

FROM classified

GROUP BY
      r_w
    , bank_name_main

ORDER BY
      r_w
    , bank_name_main;
