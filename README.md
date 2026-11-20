
USE ALM;
SET NOCOUNT ON;

DECLARE @DateFrom date = '2026-11-01';
DECLARE @DateTo date = '2026-11-18';
DECLARE @threshold decimal(18,2) = 200000000;

;WITH base AS (
    SELECT
        CASE
            WHEN e.CLI_ID IS NOT NULL THEN N'Действующий'
            ELSE N'Новый'
        END AS client_type,

        CASE
            WHEN t.spec_cat IN ('BR','SR','FU') THEN N'Market'
            WHEN t.transaction_type = N'Наличные' THEN N'Наличные'
            ELSE t.bank_name_main
        END AS bank,

        CASE
            WHEN t.spec_cat IN ('BR','SR','FU') THEN 'market'
            WHEN t.transaction_type = N'Наличные' THEN 'cash'
            ELSE 'bank'
        END AS bank_type,

        ISNULL(t.gr_size, N'Без gr_size') AS gr_size,

        t.direction_txt,
        a.amt_ns,
        a.amt_sr,
        a.amt_dvs,
        a.amt_ns + a.amt_sr + a.amt_dvs AS amt

    FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

    INNER JOIN #pay_participants p
        ON p.CLI_ID = t.cli_id

    LEFT JOIN #existing_clients e
        ON e.CLI_ID = t.cli_id

    CROSS APPLY (
        VALUES (
            ISNULL(t.[распознано_НС], 0),
            ISNULL(t.[распознано_СР], 0),
            ISNULL(t.[Распознано_ДВС], 0)
        )
    ) a(amt_ns, amt_sr, amt_dvs)

    WHERE t.dt_rep >= @DateFrom
      AND t.dt_rep < DATEADD(day, 1, @DateTo)

      AND t.direction_type <> N'Ошибка'
      AND t.[целевая категория] <> 1
      AND t.transaction_type <> N'Внутренние переводы'
      AND t.transit_max_id IS NULL
      AND t.bank_name_main <> N'Без перевода'
      AND t.cli_biz IN (N'Розница', N'ДЧБО')

      AND (
          t.spec_cat NOT IN ('BR','SR','FU')
          OR (
              t.spec_cat IN ('BR','SR','FU')
              AND t.transaction_type <> N'Наличные'
          )
      )
),

flows AS (
    SELECT
        *,
        CASE
            WHEN direction_txt = N'входящие'
              OR (direction_txt = N'смешанные' AND amt > 0)
                THEN 1 ELSE 0
        END AS is_incoming,

        CASE
            WHEN direction_txt = N'исходящие'
              OR (direction_txt = N'смешанные' AND amt < 0)
                THEN 1 ELSE 0
        END AS is_outgoing
    FROM base
),

by_bank AS (
    SELECT
        client_type,
        bank,
        bank_type,
        gr_size,

        SUM(CASE WHEN is_incoming = 1 THEN amt_ns ELSE 0 END) AS incoming_ns,
        SUM(CASE WHEN is_outgoing = 1 THEN amt_ns ELSE 0 END) AS outgoing_ns,

        SUM(CASE WHEN is_incoming = 1 THEN amt_sr ELSE 0 END) AS incoming_sr,
        SUM(CASE WHEN is_outgoing = 1 THEN amt_sr ELSE 0 END) AS outgoing_sr,

        SUM(CASE WHEN is_incoming = 1 THEN amt_dvs ELSE 0 END) AS incoming_dvs,
        SUM(CASE WHEN is_outgoing = 1 THEN amt_dvs ELSE 0 END) AS outgoing_dvs

    FROM flows
    WHERE is_incoming = 1 OR is_outgoing = 1
    GROUP BY client_type, bank, bank_type, gr_size
),

totals AS (
    SELECT
        *,
        SUM(incoming_ns + incoming_sr + incoming_dvs)
            OVER (PARTITION BY bank, bank_type) AS bank_incoming,

        SUM(outgoing_ns + outgoing_sr + outgoing_dvs)
            OVER (PARTITION BY bank, bank_type) AS bank_outgoing
    FROM by_bank
),

classified AS (
    SELECT
        client_type,
        gr_size,

        CASE
            WHEN bank_type = 'bank'
             AND ABS(bank_incoming) <= @threshold
             AND ABS(bank_outgoing) <= @threshold
                THEN N'else'
            ELSE bank
        END AS bank_name_main,

        v.raspoznano_type,
        v.incoming,
        v.outgoing

    FROM totals

    CROSS APPLY (
        VALUES
            (N'НС', incoming_ns, outgoing_ns),
            (N'Вклады', incoming_sr, outgoing_sr),
            (N'ДВС', incoming_dvs, outgoing_dvs)
    ) v(raspoznano_type, incoming, outgoing)
)

SELECT
    client_type,
    bank_name_main,
    gr_size,
    raspoznano_type,

    SUM(incoming) / 1000000000.0 AS incoming,
    SUM(outgoing) / 1000000000.0 AS outgoing,
    SUM(incoming + outgoing) / 1000000000.0 AS saldo

FROM classified

GROUP BY
    client_type,
    bank_name_main,
    gr_size,
    raspoznano_type

ORDER BY
    client_type,
    bank_name_main,
    gr_size,
    raspoznano_type

OPTION (MAXDOP 1, RECOMPILE);
