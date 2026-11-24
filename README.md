
USE ALM;
SET NOCOUNT ON;

DECLARE @DateBase date = '2026-10-31';
DECLARE @DateFrom date = '2026-11-01';
DECLARE @DateTo date = '2026-11-18';
DECLARE @HistoryFrom date = DATEADD(day, -7, @DateFrom);

DECLARE @threshold decimal(18,2) = 200000000;
DECLARE @d date = @HistoryFrom;

DROP TABLE IF EXISTS #r1_existing;
DROP TABLE IF EXISTS #r1_triggers;
DROP TABLE IF EXISTS #r1_closed;
DROP TABLE IF EXISTS #r1_messages;
DROP TABLE IF EXISTS #r1_sms_windows;
DROP TABLE IF EXISTS #r1_push_windows;
DROP TABLE IF EXISTS #r1_clients;
DROP TABLE IF EXISTS #r1_flags_day;
DROP TABLE IF EXISTS #r1_payments;


/* Действующие клиенты на 31 октября */

SELECT
    t.CLI_ID AS cli_id

INTO #r1_existing

FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

WHERE t.dt_rep = @DateBase
  AND t.block_name = N'Привлечение ФЛ'
  AND t.section_name IN (
      N'Срочные',
      N'Накопительный счёт',
      N'До востребования'
  )
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.CLI_ID IS NOT NULL
  AND t.out_rub IS NOT NULL
  AND t.out_rub >= 0

GROUP BY t.CLI_ID
HAVING SUM(t.out_rub) > 0

OPTION (MAXDOP 1, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_r1_existing
ON #r1_existing (cli_id);


/* Переводы, способные запустить СМС или пуш */

SELECT DISTINCT
    t.cli_id,
    CAST(t.dt_rep AS date) AS dt

INTO #r1_triggers

FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

WHERE t.dt_rep >= @HistoryFrom
  AND t.dt_rep < DATEADD(day, 1, @DateTo)

  AND t.cli_id IS NOT NULL
  AND t.cli_biz = N'Розница'

  AND t.direction_type = N'Переводы себе'
  AND t.transaction_type = N'СБП-Перевод'
  AND t.direction = '-1'
  AND t.amount_out <= -1000000

  AND t.[целевая категория] = 1
  AND t.transit_max_id IS NULL
  AND t.bank_name_main <> N'Без перевода'

  AND (
      t.spec_cat NOT IN ('BR','SR','FU')
      OR (
          t.spec_cat IN ('BR','SR','FU')
          AND t.transaction_type <> N'Наличные'
      )
  )

OPTION (MAXDOP 1, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_r1_triggers
ON #r1_triggers (cli_id, dt);


/* Закрытия срочных вкладов */

SELECT
    a.cli_id,
    CAST(a.DT_CLOSE AS date) AS dt_close

INTO #r1_closed

FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)

INNER JOIN (
    SELECT DISTINCT cli_id
    FROM #r1_triggers
) c
    ON c.cli_id = a.cli_id

WHERE a.DT_CLOSE >= DATEADD(day, -2, @HistoryFrom)
  AND a.DT_CLOSE < DATEADD(day, 1, @DateTo)

  AND a.cli_short_name = N'ФЛ'
  AND a.cur = 'RUR'

  AND a.PROD_NAME NOT IN (
      N'Накопительный счёт',
      N'Накопительный счёт Ультра',
      N'Агентские эскроу ФЛ по ставке КС+спред',
      N'Агентские эскроу ФЛ по ставке КС/2',
      N'Спец. банк.счёт',
      N'Спец. банк. счёт',
      N'Залоговый',
      N'Эскроу',
      N'Депозит ФЛ (суррогатный договор для счета Новой Афины)',
      N'Тип договора не определен'
  )

GROUP BY
    a.cli_id,
    CAST(a.DT_CLOSE AS date)

OPTION (MAXDOP 1, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_r1_closed
ON #r1_closed (cli_id, dt_close);


/* Фактические СМС, пуши и просмотры.
   Одна строка на CLI_ID и календарный день */

SELECT
    m.cli_id,
    v.dt,

    MAX(v.sms_sent) AS sms_sent,
    MAX(v.push_created) AS push_created,
    MAX(v.push_viewed) AS push_viewed,
    MAX(v.push_accepted) AS push_accepted

INTO #r1_messages

FROM ALM.ehd.VW_promocode_FL_list m WITH (NOLOCK)

CROSS APPLY (
    VALUES
        (
            CAST(m.msgbegindate AS date),

            CASE WHEN LOWER(m.message_type) = N'sms'
                THEN 1 ELSE 0 END,

            CASE WHEN LOWER(m.message_type) = N'push'
                THEN 1 ELSE 0 END,

            0,
            0
        ),
        (
            CAST(m.dt_view AS date),

            0,
            0,

            CASE WHEN LOWER(m.message_type) = N'push'
                THEN 1 ELSE 0 END,

            0
        ),
        (
            CAST(m.dt_accept AS date),

            0,
            0,
            0,

            CASE WHEN LOWER(m.message_type) = N'push'
                THEN 1 ELSE 0 END
        )
) v(
    dt,
    sms_sent,
    push_created,
    push_viewed,
    push_accepted
)

WHERE m.cli_id IS NOT NULL
  AND m.msgbegindate >= @HistoryFrom
  AND m.msgbegindate < DATEADD(day, 1, @DateTo)
  AND v.dt BETWEEN @HistoryFrom AND @DateTo

GROUP BY
    m.cli_id,
    v.dt

OPTION (MAXDOP 1, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_r1_messages
ON #r1_messages (cli_id, dt);


/* Потенциальные окна СМС.
   Следующее окно начинается не раньше чем через 7 дней.
   Предыдущая фактическая СМС блокирует повторное окно. */

SELECT TOP (0)
    cli_id,
    dt AS dt_from,
    dt AS dt_to

INTO #r1_sms_windows

FROM #r1_triggers;

CREATE UNIQUE CLUSTERED INDEX IX_r1_sms_windows
ON #r1_sms_windows (cli_id, dt_from);

WHILE @d <= @DateTo
BEGIN

    INSERT INTO #r1_sms_windows (
        cli_id,
        dt_from,
        dt_to
    )

    SELECT
        t.cli_id,
        @d,
        DATEADD(day, 6, @d)

    FROM #r1_triggers t

    WHERE t.dt = @d

      AND NOT EXISTS (
          SELECT 1
          FROM #r1_sms_windows w
          WHERE w.cli_id = t.cli_id
            AND @d BETWEEN w.dt_from AND w.dt_to
      )

      AND NOT EXISTS (
          SELECT 1
          FROM #r1_messages m
          WHERE m.cli_id = t.cli_id
            AND m.sms_sent = 1
            AND m.dt BETWEEN DATEADD(day, -6, @d)
                         AND DATEADD(day, -1, @d)
      );

    SET @d = DATEADD(day, 1, @d);

END;


/* Потенциальные окна пуша:
   закрытие вклада + перевод в течение 2 дней */

SELECT
    t.cli_id,
    t.dt AS dt_from,
    MAX(DATEADD(day, 2, c.dt_close)) AS dt_to

INTO #r1_push_windows

FROM #r1_triggers t

INNER JOIN #r1_closed c
    ON c.cli_id = t.cli_id
   AND t.dt BETWEEN c.dt_close
                AND DATEADD(day, 2, c.dt_close)

GROUP BY
    t.cli_id,
    t.dt;

CREATE UNIQUE CLUSTERED INDEX IX_r1_push_windows
ON #r1_push_windows (cli_id, dt_from);


/* Потенциальное право ИЛИ факт уведомления */

SELECT
    ids.cli_id,

    CASE
        WHEN e.cli_id IS NOT NULL
            THEN N'Действующий'
        ELSE N'Новый'
    END AS client_type,

    CASE WHEN EXISTS (
        SELECT 1
        FROM #r1_sms_windows w
        WHERE w.cli_id = ids.cli_id
          AND w.dt_to >= @DateFrom
    ) THEN 1 ELSE 0 END AS sms_eligible_ever,

    CASE WHEN EXISTS (
        SELECT 1
        FROM #r1_push_windows w
        WHERE w.cli_id = ids.cli_id
          AND w.dt_to >= @DateFrom
    ) THEN 1 ELSE 0 END AS push_eligible_ever,

    CASE WHEN EXISTS (
        SELECT 1
        FROM #r1_messages m
        WHERE m.cli_id = ids.cli_id
          AND m.sms_sent = 1
          AND m.dt >= DATEADD(day, -6, @DateFrom)
    ) THEN 1 ELSE 0 END AS sms_fact_ever,

    CASE WHEN EXISTS (
        SELECT 1
        FROM #r1_messages m
        WHERE m.cli_id = ids.cli_id
          AND m.push_created = 1
          AND m.dt >= DATEADD(day, -2, @DateFrom)
    ) THEN 1 ELSE 0 END AS push_fact_ever,

    CASE WHEN EXISTS (
        SELECT 1
        FROM #r1_messages m
        WHERE m.cli_id = ids.cli_id
          AND m.push_viewed = 1
    ) THEN 1 ELSE 0 END AS push_viewed_ever

INTO #r1_clients

FROM (
    SELECT cli_id
    FROM #r1_sms_windows
    WHERE dt_to >= @DateFrom

    UNION

    SELECT cli_id
    FROM #r1_push_windows
    WHERE dt_to >= @DateFrom

    UNION

    SELECT cli_id
    FROM #r1_messages
    WHERE (
        sms_sent = 1
        AND dt >= DATEADD(day, -6, @DateFrom)
    )
    OR (
        push_created = 1
        AND dt >= DATEADD(day, -2, @DateFrom)
    )
    OR (
        push_viewed = 1
        AND dt >= @DateFrom
    )
) ids

LEFT JOIN #r1_existing e
    ON e.cli_id = ids.cli_id;

CREATE UNIQUE CLUSTERED INDEX IX_r1_clients
ON #r1_clients (cli_id);


/* Подневные флаги клиентов.
   Раскрываем только даты внутри окон.
   Одна строка на CLI_ID и дату. */

;WITH nums AS (
    SELECT n
    FROM (
        VALUES (0),(1),(2),(3),(4),(5),(6)
    ) v(n)
),

flags AS (

    /* Потенциальное окно СМС */

    SELECT
        w.cli_id,
        DATEADD(day, n.n, w.dt_from) AS dt,

        1 AS sms_window,
        0 AS push_window,
        0 AS sms_offer_window,
        0 AS push_view_day

    FROM #r1_sms_windows w
    CROSS JOIN nums n

    WHERE DATEADD(day, n.n, w.dt_from) <= w.dt_to

    UNION ALL

    /* Потенциальное окно пуша */

    SELECT
        w.cli_id,
        DATEADD(day, n.n, w.dt_from),

        0, 1, 0, 0

    FROM #r1_push_windows w
    CROSS JOIN nums n

    WHERE DATEADD(day, n.n, w.dt_from) <= w.dt_to

    UNION ALL

    /* Окно фактической СМС */

    SELECT
        m.cli_id,
        DATEADD(day, n.n, m.dt),

        0, 0, 1, 0

    FROM #r1_messages m
    CROSS JOIN nums n

    WHERE m.sms_sent = 1

    UNION ALL

    /* Дата просмотра пуша */

    SELECT
        m.cli_id,
        m.dt,

        0, 0, 0, 1

    FROM #r1_messages m

    WHERE m.push_viewed = 1
)

SELECT
    f.cli_id,
    f.dt,

    MAX(f.sms_window) AS sms_window,
    MAX(f.push_window) AS push_window,
    MAX(f.sms_offer_window) AS sms_offer_window,
    MAX(f.push_view_day) AS push_view_day

INTO #r1_flags_day

FROM flags f

INNER JOIN #r1_clients c
    ON c.cli_id = f.cli_id

WHERE f.dt BETWEEN @DateFrom AND @DateTo

GROUP BY
    f.cli_id,
    f.dt

OPTION (MAXDOP 1, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_r1_flags_day
ON #r1_flags_day (cli_id, dt);


/* ВСЕ платежи.
   Не только СБП и не только платежи от 1 млн.
   Клиенты вне удержания получают нулевые флаги.
   Сразу агрегируем по дате и флагам. */

;WITH base AS (
    SELECT
        CAST(t.dt_rep AS date) AS dt_rep,

        CASE
            WHEN e.cli_id IS NOT NULL
                THEN N'Действующий'
            ELSE N'Новый'
        END AS client_type,

        CASE
            WHEN t.spec_cat IN ('BR','SR','FU')
                THEN N'Market'

            WHEN t.transaction_type = N'Наличные'
                THEN N'Наличные'

            ELSE t.bank_name_main
        END AS bank,

        CASE
            WHEN t.spec_cat IN ('BR','SR','FU')
                THEN 'market'

            WHEN t.transaction_type = N'Наличные'
                THEN 'cash'

            ELSE 'bank'
        END AS bank_type,

        ISNULL(t.gr_size, N'Без gr_size') AS gr_size,

        ISNULL(c.sms_eligible_ever, 0)
            AS sms_eligible_ever,

        ISNULL(c.push_eligible_ever, 0)
            AS push_eligible_ever,

        ISNULL(c.sms_fact_ever, 0)
            AS sms_fact_ever,

        ISNULL(c.push_fact_ever, 0)
            AS push_fact_ever,

        ISNULL(c.push_viewed_ever, 0)
            AS push_viewed_ever,

        CASE
            WHEN t.direction_type = N'Переводы себе'
             AND t.transaction_type = N'СБП-Перевод'
             AND t.direction = '-1'
             AND t.amount_out <= -1000000
             AND t.cli_biz = N'Розница'
                THEN 1
            ELSE 0
        END AS trigger_payment,

        ISNULL(f.sms_window, 0) AS sms_window,
        ISNULL(f.push_window, 0) AS push_window,

        ISNULL(f.sms_offer_window, 0)
            AS sms_offer_window,

        ISNULL(f.push_view_day, 0)
            AS push_view_day,

        a.amt_ns,
        a.amt_sr,
        a.amt_dvs,

        CASE
            WHEN t.direction_txt = N'входящие'
              OR (
                  t.direction_txt = N'смешанные'
                  AND a.amt > 0
              )
                THEN 1
            ELSE 0
        END AS is_incoming,

        CASE
            WHEN t.direction_txt = N'исходящие'
              OR (
                  t.direction_txt = N'смешанные'
                  AND a.amt < 0
              )
                THEN 1
            ELSE 0
        END AS is_outgoing

    FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

    LEFT JOIN #r1_existing e
        ON e.cli_id = t.cli_id

    LEFT JOIN #r1_clients c
        ON c.cli_id = t.cli_id

    LEFT JOIN #r1_flags_day f
        ON f.cli_id = t.cli_id
       AND f.dt = CAST(t.dt_rep AS date)

    CROSS APPLY (
        VALUES (
            CAST(
                ISNULL(t.[распознано_НС], 0)
                AS decimal(28,6)
            ),

            CAST(
                ISNULL(t.[распознано_СР], 0)
                AS decimal(28,6)
            ),

            CAST(
                ISNULL(t.[Распознано_ДВС], 0)
                AS decimal(28,6)
            )
        )
    ) v(amt_ns, amt_sr, amt_dvs)

    CROSS APPLY (
        VALUES (
            v.amt_ns,
            v.amt_sr,
            v.amt_dvs,

            v.amt_ns
            + v.amt_sr
            + v.amt_dvs
        )
    ) a(amt_ns, amt_sr, amt_dvs, amt)

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
    dt_rep,
    client_type,
    bank,
    bank_type,
    gr_size,

    sms_eligible_ever,
    push_eligible_ever,
    sms_fact_ever,
    push_fact_ever,
    push_viewed_ever,

    trigger_payment,
    sms_window,
    push_window,
    sms_offer_window,
    push_view_day,

    SUM(CASE
        WHEN is_incoming = 1
            THEN amt_ns ELSE 0
    END) AS incoming_ns,

    SUM(CASE
        WHEN is_outgoing = 1
            THEN amt_ns ELSE 0
    END) AS outgoing_ns,

    SUM(CASE
        WHEN is_incoming = 1
            THEN amt_sr ELSE 0
    END) AS incoming_sr,

    SUM(CASE
        WHEN is_outgoing = 1
            THEN amt_sr ELSE 0
    END) AS outgoing_sr,

    SUM(CASE
        WHEN is_incoming = 1
            THEN amt_dvs ELSE 0
    END) AS incoming_dvs,

    SUM(CASE
        WHEN is_outgoing = 1
            THEN amt_dvs ELSE 0
    END) AS outgoing_dvs

INTO #r1_payments

FROM base

WHERE is_incoming = 1
   OR is_outgoing = 1

GROUP BY
    dt_rep,
    client_type,
    bank,
    bank_type,
    gr_size,

    sms_eligible_ever,
    push_eligible_ever,
    sms_fact_ever,
    push_fact_ever,
    push_viewed_ever,

    trigger_payment,
    sms_window,
    push_window,
    sms_offer_window,
    push_view_day

OPTION (MAXDOP 1, RECOMPILE);


/* Итог = банк + gr_size + виды средств + флаги.
   Порог банка считаем по всем платежам. */

;WITH bank_totals AS (
    SELECT
        bank,
        bank_type,

        SUM(
            ISNULL(incoming_ns, 0)
          + ISNULL(incoming_sr, 0)
          + ISNULL(incoming_dvs, 0)
        ) AS bank_incoming,

        SUM(
            ISNULL(outgoing_ns, 0)
          + ISNULL(outgoing_sr, 0)
          + ISNULL(outgoing_dvs, 0)
        ) AS bank_outgoing

    FROM #r1_payments

    GROUP BY
        bank,
        bank_type
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
        END AS bank_name_main

    FROM #r1_payments p

    INNER JOIN bank_totals b
        ON b.bank = p.bank
       AND b.bank_type = p.bank_type
)

SELECT
    p.dt_rep,
    p.client_type,
    p.bank_name_main,
    p.gr_size,
    v.raspoznano_type,

    CASE
        WHEN p.sms_eligible_ever = 1
         AND p.push_eligible_ever = 1
            THEN N'SMS + Push'

        WHEN p.sms_eligible_ever = 1
            THEN N'SMS'

        WHEN p.push_eligible_ever = 1
            THEN N'Push'

        ELSE N'Нет'
    END AS eligible_group,

    CASE
        WHEN p.sms_fact_ever = 1
         AND p.push_fact_ever = 1
            THEN N'SMS + Push'

        WHEN p.sms_fact_ever = 1
            THEN N'SMS'

        WHEN p.push_fact_ever = 1
            THEN N'Push'

        ELSE N'Нет'
    END AS fact_group,

    p.push_viewed_ever,
    p.sms_window,
    p.push_window,
    p.sms_offer_window,
    p.push_view_day,
    p.trigger_payment,

    SUM(v.incoming) / 1000000000.0 AS incoming,

    SUM(v.outgoing) / 1000000000.0 AS outgoing,

    SUM(
        v.incoming + v.outgoing
    ) / 1000000000.0 AS saldo

FROM classified p

CROSS APPLY (
    VALUES
        (
            N'НС',
            p.incoming_ns,
            p.outgoing_ns
        ),
        (
            N'Вклады',
            p.incoming_sr,
            p.outgoing_sr
        ),
        (
            N'ДВС',
            p.incoming_dvs,
            p.outgoing_dvs
        )
) v(raspoznano_type, incoming, outgoing)

GROUP BY
    p.dt_rep,
    p.client_type,
    p.bank_name_main,
    p.gr_size,
    v.raspoznano_type,

    p.sms_eligible_ever,
    p.push_eligible_ever,
    p.sms_fact_ever,
    p.push_fact_ever,
    p.push_viewed_ever,

    p.sms_window,
    p.push_window,
    p.sms_offer_window,
    p.push_view_day,
    p.trigger_payment

ORDER BY
    p.dt_rep,
    p.client_type,
    p.bank_name_main,
    p.gr_size,
    v.raspoznano_type;


/* Реестр уникальных клиентов и флагов */

SELECT
    client_type,

    COUNT(*) AS clients,

    SUM(sms_eligible_ever) AS sms_eligible_clients,
    SUM(push_eligible_ever) AS push_eligible_clients,

    SUM(sms_fact_ever) AS sms_fact_clients,
    SUM(push_fact_ever) AS push_fact_clients,

    SUM(push_viewed_ever) AS push_viewed_clients

FROM #r1_clients

GROUP BY client_type;
