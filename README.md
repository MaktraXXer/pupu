USE ALM;
SET NOCOUNT ON;

/* ============================================================
   PARAMETERS
   ============================================================ */

DECLARE @dt_to_mar date = '2026-03-24';
DECLARE @dt_to_sep date = '2025-10-24';
DECLARE @dt_to_oct date = '2026-10-24';

DECLARE @threshold decimal(18,2) = 200000000;


/* ============================================================
   1. PERIODS
   ============================================================ */

IF OBJECT_ID('tempdb..#periods') IS NOT NULL DROP TABLE #periods;

CREATE TABLE #periods
(
    period_id int PRIMARY KEY,
    dt_from date,
    dt_to date,
    r_w varchar(30)
);

INSERT INTO #periods
VALUES
(1, '2026-03-01', @dt_to_mar,
    '2026 01.03-' + CONVERT(varchar(5), @dt_to_mar, 104)),

(2, '2025-10-01', @dt_to_sep,
    '2025 01.10-' + CONVERT(varchar(5), @dt_to_sep, 104)),

(3, '2026-10-01', @dt_to_oct,
    '2026 01.10-' + CONVERT(varchar(5), @dt_to_oct, 104));


/* ============================================================
   2. PREMIUM CLIENTS

   Определяем премиальных клиентов отдельно для каждого
   периода на дату его окончания.

   DISTINCT исключает размножение платежей из-за нескольких
   записей атрибута.
   ============================================================ */

IF OBJECT_ID('tempdb..#premium_clients') IS NOT NULL
    DROP TABLE #premium_clients;

SELECT DISTINCT
      p.period_id
    , a.CLI_ID AS cli_id

INTO #premium_clients

FROM ALM.ehd.attr_cli_stat_sal a WITH (NOLOCK)

INNER JOIN #periods p
    ON p.dt_to BETWEEN a.DT_FROM AND a.DT_TO

WHERE a.CLI_ATTR_CODE = 'PREMIUM_PACKAGE'
  AND a.CLI_ID IS NOT NULL;

CREATE UNIQUE CLUSTERED INDEX IX_premium_clients
    ON #premium_clients(period_id, cli_id);


/* ============================================================
   3. AGGREGATED FLOWS

   Из агрегатов забираем ДЧБО и всю Розницу.
   Деления на премиум и масс здесь ещё нет.

   Сохраняем исходные фильтры и классификацию банков.
   ============================================================ */

IF OBJECT_ID('tempdb..#agg_bank') IS NOT NULL DROP TABLE #agg_bank;

;WITH flows AS
(
    SELECT
          p.period_id
        , t.cli_biz AS segment

        , CASE
            WHEN t.spec_cat IN ('BR','SR','FU')
                THEN N'Market'
            WHEN t.transaction_type = N'Наличные'
                THEN N'Наличные'
            ELSE t.bank_name_main
          END AS bank

        , CASE
            WHEN t.spec_cat IN ('BR','SR','FU') THEN 'market'
            WHEN t.transaction_type = N'Наличные' THEN 'cash'
            ELSE 'bank'
          END AS bank_type

        , t.direction_txt

        , ISNULL(t.[Распознано_НС],0)
        + ISNULL(t.[Распознано_СР],0)
        + ISNULL(t.[Распознано_ДВС],0) AS amt

    FROM ALM.ehd.VW_transfers_FL_AGG_tab t WITH (NOLOCK)

    INNER JOIN #periods p
        ON t.dt_rep >= p.dt_from
       AND t.dt_rep < DATEADD(day,1,p.dt_to)

    WHERE t.direction_type <> N'Ошибка'
      AND t.[целевая категория] = 1
      AND t.transaction_type <> N'Внутренние переводы'
      AND t.transit_flag = N'Не транзит'
      AND t.cli_biz IN (N'Розница',N'ДЧБО')

      AND
      (
          t.spec_cat NOT IN ('BR','SR','FU')
          OR
          (
              t.spec_cat IN ('BR','SR','FU')
              AND t.transaction_type <> N'Наличные'
          )
      )
)

SELECT
      period_id
    , segment
    , bank
    , bank_type

    , SUM(
        CASE
            WHEN direction_txt = N'входящие'
              OR (direction_txt = N'смешанные' AND amt > 0)
                THEN amt
            ELSE 0
        END
      ) AS incoming

    , SUM(
        CASE
            WHEN direction_txt = N'исходящие'
              OR (direction_txt = N'смешанные' AND amt < 0)
                THEN amt
            ELSE 0
        END
      ) AS outgoing

INTO #agg_bank

FROM flows

GROUP BY
      period_id
    , segment
    , bank
    , bank_type

OPTION (RECOMPILE);


/* ============================================================
   4. PREMIUM FLOWS FROM DETAILS

   Самое важное:

   1. Только cli_biz = Розница.
   2. Только клиенты из #premium_clients.
   3. Только платежи соответствующего периода.

   Массовых клиентов отдельно НЕ выгружаем.

   Фильтры и расчёт потоков повторяют агрегаты.
   ============================================================ */

IF OBJECT_ID('tempdb..#premium_bank') IS NOT NULL
    DROP TABLE #premium_bank;

;WITH flows AS
(
    SELECT
          p.period_id

        , CASE
            WHEN t.spec_cat IN ('BR','SR','FU')
                THEN N'Market'
            WHEN t.transaction_type = N'Наличные'
                THEN N'Наличные'
            ELSE t.bank_name_main
          END AS bank

        , CASE
            WHEN t.spec_cat IN ('BR','SR','FU') THEN 'market'
            WHEN t.transaction_type = N'Наличные' THEN 'cash'
            ELSE 'bank'
          END AS bank_type

        , t.direction_txt

        , ISNULL(t.[Распознано_НС],0)
        + ISNULL(t.[Распознано_СР],0)
        + ISNULL(t.[Распознано_ДВС],0) AS amt

    FROM #premium_clients pc

    INNER JOIN #periods p
        ON p.period_id = pc.period_id

    INNER JOIN ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)
        ON t.cli_id = pc.cli_id
       AND t.dt_rep >= p.dt_from
       AND t.dt_rep < DATEADD(day,1,p.dt_to)

    WHERE t.cli_biz = N'Розница'

      AND t.direction_type <> N'Ошибка'
      AND t.[целевая категория] = 1
      AND t.transaction_type <> N'Внутренние переводы'
      AND t.transit_max_id IS NULL

      AND
      (
          t.spec_cat NOT IN ('BR','SR','FU')
          OR
          (
              t.spec_cat IN ('BR','SR','FU')
              AND t.transaction_type <> N'Наличные'
          )
      )
)

SELECT
      period_id
    , bank
    , bank_type

    , SUM(
        CASE
            WHEN direction_txt = N'входящие'
              OR (direction_txt = N'смешанные' AND amt > 0)
                THEN amt
            ELSE 0
        END
      ) AS incoming

    , SUM(
        CASE
            WHEN direction_txt = N'исходящие'
              OR (direction_txt = N'смешанные' AND amt < 0)
                THEN amt
            ELSE 0
        END
      ) AS outgoing

INTO #premium_bank

FROM flows

GROUP BY
      period_id
    , bank
    , bank_type

OPTION (RECOMPILE);


/* ============================================================
   5. BANK THRESHOLD

   Определяем маленькие банки по исходному общему потоку:
   ДЧБО + вся Розница.

   Порог применяем ДО выделения премиума,
   чтобы банк не менял категорию только из-за разделения
   клиентских сегментов.
   ============================================================ */

IF OBJECT_ID('tempdb..#bank_threshold') IS NOT NULL
    DROP TABLE #bank_threshold;

SELECT
      period_id
    , bank
    , bank_type
    , SUM(incoming) AS incoming
    , SUM(outgoing) AS outgoing

INTO #bank_threshold

FROM #agg_bank

GROUP BY
      period_id
    , bank
    , bank_type;


/* ============================================================
   6. FINAL RESULT

   ДЧБО    = агрегаты ДЧБО
   Премиум = детализация Розницы с атрибутом PREMIUM_PACKAGE
   Масс    = агрегаты Розницы - детализация Премиума
   ============================================================ */

;WITH segmented AS
(
    /* ДЧБО */
    SELECT
          period_id
        , N'ДЧБО' AS segment
        , bank
        , bank_type
        , incoming
        , outgoing

    FROM #agg_bank
    WHERE segment = N'ДЧБО'


    UNION ALL


    /* Премиум */
    SELECT
          period_id
        , N'Премиум'
        , bank
        , bank_type
        , incoming
        , outgoing

    FROM #premium_bank


    UNION ALL


    /* Масс = Розница - Премиум */
    SELECT
          COALESCE(r.period_id,pr.period_id)
        , N'Масс'
        , COALESCE(r.bank,pr.bank)
        , COALESCE(r.bank_type,pr.bank_type)

        , ISNULL(r.incoming,0) - ISNULL(pr.incoming,0)
        , ISNULL(r.outgoing,0) - ISNULL(pr.outgoing,0)

    FROM
    (
        SELECT *
        FROM #agg_bank
        WHERE segment = N'Розница'
    ) r

    FULL OUTER JOIN #premium_bank pr
        ON pr.period_id = r.period_id
       AND pr.bank_type = r.bank_type
       AND
       (
           pr.bank = r.bank
           OR (pr.bank IS NULL AND r.bank IS NULL)
       )
),

classified AS
(
    SELECT
          s.*

        , CASE
            WHEN s.bank_type = 'bank'
             AND ABS(b.incoming) <= @threshold
             AND ABS(b.outgoing) <= @threshold
                THEN N'else'
            ELSE s.bank
          END AS final_bank

    FROM segmented s

    LEFT JOIN #bank_threshold b
        ON b.period_id = s.period_id
       AND b.bank_type = s.bank_type
       AND
       (
           b.bank = s.bank
           OR (b.bank IS NULL AND s.bank IS NULL)
       )
)

SELECT
      p.r_w
    , c.segment
    , c.final_bank AS bank_name_main

    , SUM(c.incoming) / 1000000000.0 AS incoming
    , SUM(c.outgoing) / 1000000000.0 AS outgoing
    , SUM(c.incoming + c.outgoing) / 1000000000.0 AS saldo

FROM classified c

INNER JOIN #periods p
    ON p.period_id = c.period_id

GROUP BY
      p.period_id
    , p.r_w
    , c.segment
    , c.final_bank

ORDER BY
      p.period_id
    , c.segment
    , c.final_bank;


/* ============================================================
   7. CONTROL

   Проверяем, нет ли банков, по которым детализация
   премиума превышает весь агрегированный поток Розницы.

   Если результат пустой - таких противоречий нет.

   Это диагностическая проверка, не замена сверки
   витрин детализации и агрегатов.
   ============================================================ */

SELECT
      p.r_w
    , pr.bank
    , pr.bank_type

    , ISNULL(r.incoming,0) AS retail_incoming
    , pr.incoming AS premium_incoming

    , ISNULL(r.outgoing,0) AS retail_outgoing
    , pr.outgoing AS premium_outgoing

FROM #premium_bank pr

INNER JOIN #periods p
    ON p.period_id = pr.period_id

LEFT JOIN #agg_bank r
    ON r.period_id = pr.period_id
   AND r.segment = N'Розница'
   AND r.bank_type = pr.bank_type
   AND
   (
       r.bank = pr.bank
       OR (r.bank IS NULL AND pr.bank IS NULL)
   )

WHERE
       pr.incoming > ISNULL(r.incoming,0) + 1
    OR pr.outgoing < ISNULL(r.outgoing,0) - 1

ORDER BY
      p.period_id
    , pr.bank;


/* ============================================================
   8. CLEANUP

   После выполнения очищаем промежуточные таблицы.
   ============================================================ */

DROP TABLE #bank_threshold;
DROP TABLE #premium_bank;
DROP TABLE #agg_bank;
DROP TABLE #premium_clients;
DROP TABLE #periods;
