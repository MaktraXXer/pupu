
USE ALM;
SET NOCOUNT ON;

DECLARE @DateFrom date = '2026-10-01';
DECLARE @DateTo date = '2026-11-04';

DECLARE @threshold decimal(18,2) = 200000000;

;WITH base AS (
    SELECT
        t.bank_name_main,
        t.spec_cat,
        t.transaction_type,
        t.direction_txt,

        ISNULL(t.gr_size, N'Без gr_size') AS gr_size,

        t.[распознано_НС]
        + t.[распознано_СР]
        + t.[Распознано_ДВС] AS amt

    FROM ALM.ehd.VW_transfers_FL_AGG_tab t WITH (NOLOCK)

    WHERE t.dt_rep >= @DateFrom
      AND t.dt_rep < DATEADD(day, 1, @DateTo)

      AND t.direction_type <> N'Ошибка'
      AND t.[целевая категория] = 1
      AND t.transaction_type <> N'Внутренние переводы'
      AND t.transit_flag = N'Не транзит'
      AND t.bank_name_main <> N'Без перевода'
      AND t.cli_biz IN (N'Розница', N'ДЧБО')
),

flows AS (
    SELECT
        gr_size,

        CASE
            WHEN spec_cat IN ('BR','SR','FU')
                THEN N'Market'
            WHEN transaction_type = N'Наличные'
                THEN N'Наличные'
            ELSE bank_name_main
        END AS bank,

        CASE
            WHEN spec_cat IN ('BR','SR','FU') THEN 'market'
            WHEN transaction_type = N'Наличные' THEN 'cash'
            ELSE 'bank'
        END AS bank_type,

        CASE
            WHEN direction_txt = N'входящие'
              OR (direction_txt = N'смешанные' AND amt > 0)
                THEN amt
            ELSE 0
        END AS incoming,

        CASE
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

/* Сначала агрегируем по банку и gr_size */

by_bank_gr AS (
    SELECT
        bank,
        bank_type,
        gr_size,
        SUM(incoming) AS incoming,
        SUM(outgoing) AS outgoing

    FROM flows

    GROUP BY
        bank,
        bank_type,
        gr_size
),

/* Затем считаем объемы банка ПО ВСЕМ gr_size */

bank_totals AS (
    SELECT
        bank,
        bank_type,
        gr_size,
        incoming,
        outgoing,

        SUM(incoming) OVER (
            PARTITION BY bank, bank_type
        ) AS bank_incoming,

        SUM(outgoing) OVER (
            PARTITION BY bank, bank_type
        ) AS bank_outgoing

    FROM by_bank_gr
),

/* Классификация банка по полному объему */

classified AS (
    SELECT
        gr_size,

        CASE
            WHEN bank_type = 'bank'
             AND ABS(bank_incoming) <= @threshold
             AND ABS(bank_outgoing) <= @threshold
                THEN N'else'
            ELSE bank
        END AS bank_name_main,

        incoming,
        outgoing

    FROM bank_totals
)

SELECT
    N'2026 01.10-04.11' AS r_w,

    bank_name_main,
    gr_size,

    SUM(incoming) / 1000000000.0 AS incoming,
    SUM(outgoing) / 1000000000.0 AS outgoing,

    SUM(incoming + outgoing) / 1000000000.0 AS saldo

FROM classified

GROUP BY
    bank_name_main,
    gr_size

ORDER BY
    bank_name_main,
    gr_size;
