
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

        t.[распознано_НС] AS amt_ns,
        t.[распознано_СР] AS amt_sr,
        t.[Распознано_ДВС] AS amt_dvs,

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

        amt_ns,
        amt_sr,
        amt_dvs,

        CASE
            WHEN amt IS NOT NULL
             AND (
                direction_txt = N'входящие'
                OR (direction_txt = N'смешанные' AND amt > 0)
             )
                THEN 1
            ELSE 0
        END AS is_incoming,

        CASE
            WHEN amt IS NOT NULL
             AND (
                direction_txt = N'исходящие'
                OR (direction_txt = N'смешанные' AND amt < 0)
             )
                THEN 1
            ELSE 0
        END AS is_outgoing

    FROM base

    WHERE spec_cat NOT IN ('BR','SR','FU')
       OR (
            spec_cat IN ('BR','SR','FU')
            AND transaction_type <> N'Наличные'
       )
),

/* Сначала агрегируем платежи банка по gr_size.
   Все три типа распознано пока находятся в одной строке. */

by_bank_gr AS (
    SELECT
        bank,
        bank_type,
        gr_size,

        SUM(CASE WHEN is_incoming = 1
            THEN amt_ns ELSE 0 END) AS incoming_ns,

        SUM(CASE WHEN is_outgoing = 1
            THEN amt_ns ELSE 0 END) AS outgoing_ns,

        SUM(CASE WHEN is_incoming = 1
            THEN amt_sr ELSE 0 END) AS incoming_sr,

        SUM(CASE WHEN is_outgoing = 1
            THEN amt_sr ELSE 0 END) AS outgoing_sr,

        SUM(CASE WHEN is_incoming = 1
            THEN amt_dvs ELSE 0 END) AS incoming_dvs,

        SUM(CASE WHEN is_outgoing = 1
            THEN amt_dvs ELSE 0 END) AS outgoing_dvs

    FROM flows

    GROUP BY
        bank,
        bank_type,
        gr_size
),

/* Считаем полные объемы банка.
   НЕТ разделения ни по gr_size, ни по типу распознано. */

bank_totals AS (
    SELECT
        g.*,

        SUM(
            ISNULL(incoming_ns, 0)
          + ISNULL(incoming_sr, 0)
          + ISNULL(incoming_dvs, 0)
        ) OVER (
            PARTITION BY bank, bank_type
        ) AS bank_incoming,

        SUM(
            ISNULL(outgoing_ns, 0)
          + ISNULL(outgoing_sr, 0)
          + ISNULL(outgoing_dvs, 0)
        ) OVER (
            PARTITION BY bank, bank_type
        ) AS bank_outgoing

    FROM by_bank_gr g
),

/* Классифицируем банк по общему объему.
   Только после этого разворачиваем три типа распознано. */

classified AS (
    SELECT
        g.gr_size,

        CASE
            WHEN g.bank_type = 'bank'
             AND ABS(g.bank_incoming) <= @threshold
             AND ABS(g.bank_outgoing) <= @threshold
                THEN N'else'
            ELSE g.bank
        END AS bank_name_main,

        v.raspoznano_type,
        v.incoming,
        v.outgoing

    FROM bank_totals g

    CROSS APPLY (
        VALUES
            (N'НС',     g.incoming_ns,  g.outgoing_ns),
            (N'Вклады', g.incoming_sr,  g.outgoing_sr),
            (N'ДВС',    g.incoming_dvs, g.outgoing_dvs)
    ) v(raspoznano_type, incoming, outgoing)
)

SELECT
    N'2026 01.10-04.11' AS r_w,

    bank_name_main,
    gr_size,
    raspoznano_type,

    SUM(incoming) / 1000000000.0 AS incoming,
    SUM(outgoing) / 1000000000.0 AS outgoing,

    SUM(incoming + outgoing) / 1000000000.0 AS saldo

FROM classified

GROUP BY
    bank_name_main,
    gr_size,
    raspoznano_type

ORDER BY
    bank_name_main,
    gr_size,
    raspoznano_type

OPTION (MAXDOP 2);
