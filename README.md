
USE ALM;
SET NOCOUNT ON;

DECLARE @DateBase date = '2026-10-31';
DECLARE @DateFrom date = '2026-11-01';
DECLARE @DateTo date = '2026-11-04';
DECLARE @threshold decimal(18,2) = 200000000;

DROP TABLE IF EXISTS #dbo_clients;
DROP TABLE IF EXISTS #test_existing;
DROP TABLE IF EXISTS #test_payments;


/* 1. Клиенты с хотя бы одним DBO-договором */

SELECT DISTINCT
    v.CLI_ID
INTO #dbo_clients
FROM ALM.ehd.VW_FL_DEPO_channels v WITH (NOLOCK)
WHERE v.CON_CHANNEL = N'DBO'
  AND v.CLI_ID IS NOT NULL
  AND v.DT_OPEN_FACT < DATEADD(day, 1, @DateTo)
OPTION (MAXDOP 1, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_dbo_clients
ON #dbo_clients (CLI_ID);


/* 2. Действующие клиенты на 31 октября */

SELECT
    t.CLI_ID
INTO #test_existing
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

INNER JOIN #dbo_clients d
    ON d.CLI_ID = t.CLI_ID

WHERE t.dt_rep = @DateBase
  AND t.block_name = N'Привлечение ФЛ'
  AND t.section_name IN (
      N'Срочные',
      N'Накопительный счёт',
      N'До востребования'
  )
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub >= 0

GROUP BY t.CLI_ID
HAVING SUM(t.out_rub) > 0

OPTION (MAXDOP 1, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_test_existing
ON #test_existing (CLI_ID);


/* 3. Платежи клиентов с DBO-договорами */

;WITH flows AS (
    SELECT
        t.cli_id,

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

        a.amt,

        CASE
            WHEN t.direction_txt = N'входящие'
              OR (t.direction_txt = N'смешанные' AND a.amt > 0)
                THEN 1 ELSE 0
        END AS is_incoming,

        CASE
            WHEN t.direction_txt = N'исходящие'
              OR (t.direction_txt = N'смешанные' AND a.amt < 0)
                THEN 1 ELSE 0
        END AS is_outgoing

    FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

    INNER JOIN #dbo_clients d
        ON d.CLI_ID = t.cli_id

    LEFT JOIN #test_existing e
        ON e.CLI_ID = t.cli_id

    CROSS APPLY (
        VALUES (
            CAST(ISNULL(t.[распознано_НС], 0) AS decimal(28,2))
          + CAST(ISNULL(t.[распознано_СР], 0) AS decimal(28,2))
          + CAST(ISNULL(t.[Распознано_ДВС], 0) AS decimal(28,2))
        )
    ) a(amt)

    WHERE t.dt_rep >= @DateFrom
      AND t.dt_rep < DATEADD(day, 1, @DateTo)
      AND t.cli_id IS NOT NULL
      AND t.direction_type <> N'Ошибка'
      AND t.[целевая категория] = 1
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
)

SELECT
    cli_id,
    client_type,
    bank,
    bank_type,
    gr_size,

    SUM(CASE WHEN is_incoming = 1
        THEN amt ELSE 0 END) AS incoming_rub,

    SUM(CASE WHEN is_outgoing = 1
        THEN amt ELSE 0 END) AS outgoing_rub,

    COUNT_BIG(CASE
        WHEN is_outgoing = 1 AND amt <> 0
        THEN 1 END) AS outgoing_count,

    SUM(CASE WHEN is_outgoing = 1
        THEN ABS(amt) ELSE 0 END) AS outgoing_abs_rub

INTO #test_payments

FROM flows

WHERE is_incoming = 1 OR is_outgoing = 1

GROUP BY
    cli_id,
    client_type,
    bank,
    bank_type,
    gr_size

OPTION (MAXDOP 1, RECOMPILE);


/* 4. Классификация банков по всем gr_size и клиентам */

;WITH bank_totals AS (
    SELECT
        bank,
        bank_type,
        SUM(incoming_rub) AS bank_incoming,
        SUM(outgoing_rub) AS bank_outgoing

    FROM #test_payments
    GROUP BY bank, bank_type
),

classified AS (
    SELECT
        p.*,

        CASE
            WHEN p.bank_type = 'bank'
             AND ABS(b.bank_incoming) <= @threshold
             AND ABS(b.bank_outgoing) <= @threshold
                THEN N'else'
            ELSE p.bank
        END AS final_bank

    FROM #test_payments p

    INNER JOIN bank_totals b
        ON b.bank = p.bank
       AND b.bank_type = p.bank_type
)

/* 5. Итоговая статистика */

SELECT
    client_type,
    final_bank AS bank_name_main,
    gr_size,

    SUM(outgoing_count) AS outgoing_operations,

    COUNT(DISTINCT CASE
        WHEN outgoing_count > 0 THEN cli_id
    END) AS outgoing_clients,

    CAST(
        SUM(outgoing_abs_rub) / 1000000000.0
        AS decimal(20,6)
    ) AS outgoing_bln,

    CAST(
        SUM(outgoing_abs_rub)
        / NULLIF(SUM(outgoing_count), 0)
        AS decimal(20,2)
    ) AS avg_payment_rub,

    CAST(
        SUM(outgoing_abs_rub)
        / NULLIF(COUNT(DISTINCT CASE
            WHEN outgoing_count > 0 THEN cli_id END), 0)
        AS decimal(20,2)
    ) AS avg_client_rub

FROM classified

GROUP BY
    client_type,
    final_bank,
    gr_size

HAVING SUM(outgoing_count) > 0

ORDER BY
    client_type,
    final_bank,
    gr_size;
