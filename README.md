
USE ALM;
SET NOCOUNT ON;

DECLARE @dt_from   date = '2026-09-01';
DECLARE @dt_before date = '2026-10-19';
DECLARE @dt_promo  date = '2026-10-20';
DECLARE @dt_to     date = '2026-11-10';

DROP TABLE IF EXISTS #clients;
DROP TABLE IF EXISTS #snap;
DROP TABLE IF EXISTS #daily;
DROP TABLE IF EXISTS #deals;
DROP TABLE IF EXISTS #ns19;
DROP TABLE IF EXISTS #attrs;
DROP TABLE IF EXISTS #tagged;


/* 1. Клиенты, имеющие на 10.11 вклад под 15%
   сроком 45–70 дней, открытый с 20 октября. */

SELECT DISTINCT
    t.CLI_ID
INTO #clients

FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

WHERE t.dt_rep = @dt_to
  AND t.block_name = N'Привлечение ФЛ'
  AND t.section_name = N'Срочные'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub > 0
  AND t.CLI_ID IS NOT NULL

  AND t.dt_open >= @dt_promo
  AND t.dt_open < DATEADD(day, 1, @dt_to)
  AND t.termdays BETWEEN 45 AND 70
  AND ABS(t.rate_con - 0.15) <= 0.0005
  AND ISNULL(t.conv, 'AT_THE_END') = 'AT_THE_END'

OPTION (MAXDOP 2, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_clients
ON #clients (CLI_ID);


/* 2. Посделочный баланс только за 19.10 и 10.11.
   Нужен для анализа источников и новых размещений. */

SELECT
    CAST(t.dt_rep AS date) AS dt_rep,
    t.CLI_ID,
    t.CON_ID,
    t.TSEGMENTNAME,
    t.PROD_NAME_res,
    t.section_name,

    CAST(t.dt_open AS date) AS dt_open,
    CAST(t.dt_close AS date) AS dt_close,

    t.rate_con,
    t.conv,
    t.termdays,

    CAST(t.out_rub AS decimal(28,2)) AS out_rub

INTO #snap

FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

INNER JOIN #clients c
    ON c.CLI_ID = t.CLI_ID

WHERE t.dt_rep IN (@dt_before, @dt_to)
  AND t.block_name = N'Привлечение ФЛ'
  AND t.section_name IN (
      N'Срочные',
      N'Накопительный счёт'
  )
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub IS NOT NULL
  AND t.out_rub >= 0

  AND (
      t.section_name <> N'Накопительный счёт'
      OR t.acc_role = N'LIAB'
  )

OPTION (MAXDOP 2, RECOMPILE);

CREATE CLUSTERED INDEX IX_snap
ON #snap (dt_rep, CLI_ID);


/* 3. Подневные остатки.
   Историю сразу агрегируем, без хранения
   всех посделочных строк за 71 день.

   Даты 19.10 и 10.11 берем из #snap,
   чтобы посделочные суммы совпадали с агрегатами. */

;WITH calendar AS (
    SELECT @dt_from AS dt_rep

    UNION ALL

    SELECT DATEADD(day, 1, dt_rep)
    FROM calendar
    WHERE dt_rep < @dt_to
),

balances AS (
    SELECT
        CAST(t.dt_rep AS date) AS dt_rep,

        SUM(CASE
            WHEN t.section_name = N'Срочные'
                THEN CAST(t.out_rub AS decimal(28,2))
            ELSE 0
        END) AS deposits_rub,

        SUM(CASE
            WHEN t.section_name = N'Накопительный счёт'
                THEN CAST(t.out_rub AS decimal(28,2))
            ELSE 0
        END) AS ns_rub

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    INNER JOIN #clients c
        ON c.CLI_ID = t.CLI_ID

    WHERE t.dt_rep >= @dt_from
      AND t.dt_rep < DATEADD(day, 1, @dt_to)

      AND t.dt_rep NOT IN (@dt_before, @dt_to)

      AND t.block_name = N'Привлечение ФЛ'
      AND t.section_name IN (
          N'Срочные',
          N'Накопительный счёт'
      )
      AND t.od_flag = 1
      AND t.cur = '810'
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0

      AND (
          t.section_name <> N'Накопительный счёт'
          OR t.acc_role = N'LIAB'
      )

    GROUP BY CAST(t.dt_rep AS date)

    UNION ALL

    SELECT
        dt_rep,

        SUM(CASE
            WHEN section_name = N'Срочные'
                THEN out_rub
            ELSE 0
        END),

        SUM(CASE
            WHEN section_name = N'Накопительный счёт'
                THEN out_rub
            ELSE 0
        END)

    FROM #snap
    GROUP BY dt_rep
)

SELECT
    c.dt_rep,
    ISNULL(b.deposits_rub, 0) AS deposits_rub,
    ISNULL(b.ns_rub, 0) AS ns_rub

INTO #daily

FROM calendar c

LEFT JOIN balances b
    ON b.dt_rep = c.dt_rep

OPTION (MAXRECURSION 0, MAXDOP 2, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_daily
ON #daily (dt_rep);


/* 4. Выходящие вклады и новые открытия.
   Все объемы берутся непосредственно из #snap.

   EXIT: на 19.10, закрытие 20.10–10.11.
   OPEN: на 10.11, открытие 20.10–10.11. */

SELECT
    CAST('EXIT' AS varchar(4)) AS deal_type,
    CLI_ID,
    CON_ID,

    MAX(TSEGMENTNAME) AS TSEGMENTNAME,
    MAX(PROD_NAME_res) AS PROD_NAME_res,
    MAX(rate_con) AS rate_con,
    MAX(conv) AS conv,
    MAX(termdays) AS termdays,
    MAX(dt_open) AS dt_open,
    MAX(dt_close) AS dt_close,

    CAST(0 AS int) AS is_15,
    SUM(out_rub) AS out_rub

INTO #deals

FROM #snap

WHERE dt_rep = @dt_before
  AND section_name = N'Срочные'
  AND dt_close BETWEEN @dt_promo AND @dt_to

GROUP BY CLI_ID, CON_ID
HAVING SUM(out_rub) > 0;


INSERT INTO #deals
SELECT
    'OPEN',
    CLI_ID,
    CON_ID,

    MAX(TSEGMENTNAME),
    MAX(PROD_NAME_res),
    MAX(rate_con),
    MAX(conv),
    MAX(termdays),
    MAX(dt_open),
    MAX(dt_close),

    MAX(CASE
        WHEN termdays BETWEEN 45 AND 70
         AND ABS(rate_con - 0.15) <= 0.0005
         AND ISNULL(conv, 'AT_THE_END') = 'AT_THE_END'
            THEN 1
        ELSE 0
    END),

    SUM(out_rub)

FROM #snap

WHERE dt_rep = @dt_to
  AND section_name = N'Срочные'
  AND dt_open BETWEEN @dt_promo AND @dt_to

GROUP BY CLI_ID, CON_ID
HAVING SUM(out_rub) > 0;

CREATE INDEX IX_deals
ON #deals (CLI_ID, deal_type);


/* 5. Накопительные счета на 19 октября.
   Все НС, без ограничений по датам открытия. */

SELECT
    CLI_ID,
    CON_ID,

    MAX(TSEGMENTNAME) AS TSEGMENTNAME,
    MAX(PROD_NAME_res) AS PROD_NAME_res,
    MAX(rate_con) AS rate_con,

    SUM(out_rub) AS out_rub

INTO #ns19

FROM #snap

WHERE dt_rep = @dt_before
  AND section_name = N'Накопительный счёт'

GROUP BY CLI_ID, CON_ID;

CREATE INDEX IX_ns19
ON #ns19 (CLI_ID);


/* 6. Надбавки для выбранных договоров.
   Берем последнюю запись каждого CON_ID.
   Отсутствие надбавок не исключает договор. */

SELECT
    x.CON_ID,

    ISNULL(TRY_CAST(a.[Нов] AS int), 0) AS nov,
    ISNULL(TRY_CAST(a.[НДП] AS int), 0) AS ndp,
    ISNULL(TRY_CAST(a.[НДМ] AS int), 0) AS ndm,

    ISNULL(TRY_CAST(a.[ПК3] AS int), 0) AS pk3,
    ISNULL(TRY_CAST(a.[ОТ1] AS int), 0) AS ot1,

    ISNULL(TRY_CAST(a.[КП3] AS int), 0) AS kp3,
    ISNULL(TRY_CAST(a.[ПК6] AS int), 0) AS pk6,
    ISNULL(TRY_CAST(a.[ПК7] AS int), 0) AS pk7

INTO #attrs

FROM (
    SELECT DISTINCT CON_ID
    FROM #deals
    WHERE CON_ID IS NOT NULL
) x

OUTER APPLY (
    SELECT TOP (1)
        a.[Нов],
        a.[НДП],
        a.[НДМ],
        a.[ПК3],
        a.[ОТ1],
        a.[КП3],
        a.[ПК6],
        a.[ПК7]

    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    WHERE a.CON_ID = CONVERT(varchar(30), x.CON_ID)

    ORDER BY
        a.DT_UPDATE DESC,
        a.loaddate DESC
) a;

CREATE UNIQUE CLUSTERED INDEX IX_attrs
ON #attrs (CON_ID);


/* 7. Категории надбавок.
   Приоритет: новые деньги, удержание, CRM.
   Каждый договор учитывается ровно один раз. */

SELECT
    d.*,

    ISNULL(a.nov, 0) AS nov,
    ISNULL(a.ndp, 0) AS ndp,
    ISNULL(a.ndm, 0) AS ndm,

    ISNULL(a.pk3, 0) AS pk3,
    ISNULL(a.ot1, 0) AS ot1,

    ISNULL(a.kp3, 0) AS kp3,
    ISNULL(a.pk6, 0) AS pk6,
    ISNULL(a.pk7, 0) AS pk7,

    CASE
        WHEN a.nov = 1 OR a.ndp = 1 OR a.ndm = 1
            THEN N'Новые деньги'

        WHEN a.pk3 = 1 OR a.ot1 = 1
            THEN N'Удержание'

        WHEN a.kp3 = 1 OR a.pk6 = 1 OR a.pk7 = 1
            THEN N'CRM'

        ELSE N'Остальные'
    END AS category

INTO #tagged

FROM #deals d

LEFT JOIN #attrs a
    ON a.CON_ID = d.CON_ID;

CREATE INDEX IX_tagged
ON #tagged (deal_type, CLI_ID);


/* РЕЗУЛЬТАТ 1. Остатки 01.09–19.10 */

SELECT
    dt_rep,
    deposits_rub,
    ns_rub,
    deposits_rub + ns_rub AS total_rub

FROM #daily
WHERE dt_rep <= @dt_before
ORDER BY dt_rep;


/* РЕЗУЛЬТАТ 2. Остатки 01.09–10.11 */

SELECT
    dt_rep,
    deposits_rub,
    ns_rub,
    deposits_rub + ns_rub AS total_rub

FROM #daily
ORDER BY dt_rep;


/* РЕЗУЛЬТАТ 3. Вклады к закрытию на 19.10 */

SELECT
    CLI_ID,
    CON_ID,
    TSEGMENTNAME,
    PROD_NAME_res,
    dt_open,
    dt_close,
    rate_con,
    conv,
    termdays,
    out_rub,

    nov AS [Нов],
    ndp AS [НДП],
    ndm AS [НДМ],
    pk3 AS [ПК3],
    ot1 AS [ОТ1],
    kp3 AS [КП3],
    pk6 AS [ПК6],
    pk7 AS [ПК7],

    category

FROM #tagged
WHERE deal_type = 'EXIT'
ORDER BY dt_close, CLI_ID, CON_ID;


/* РЕЗУЛЬТАТ 4. Все НС на 19.10 */

SELECT *
FROM #ns19
ORDER BY CLI_ID, CON_ID;


/* РЕЗУЛЬТАТ 5. Поклиентные источники на 19.10 */

;WITH exits AS (
    SELECT
        CLI_ID,

        COUNT(*) AS exit_td_count,
        SUM(out_rub) AS exit_td_sum,

        SUM(CASE WHEN category = N'Новые деньги'
            THEN 1 ELSE 0 END) AS exit_new_money_count,
        SUM(CASE WHEN category = N'Новые деньги'
            THEN out_rub ELSE 0 END) AS exit_new_money_sum,

        SUM(CASE WHEN category = N'Удержание'
            THEN 1 ELSE 0 END) AS exit_retention_count,
        SUM(CASE WHEN category = N'Удержание'
            THEN out_rub ELSE 0 END) AS exit_retention_sum,

        SUM(CASE WHEN category = N'CRM'
            THEN 1 ELSE 0 END) AS exit_crm_count,
        SUM(CASE WHEN category = N'CRM'
            THEN out_rub ELSE 0 END) AS exit_crm_sum

    FROM #tagged
    WHERE deal_type = 'EXIT'
    GROUP BY CLI_ID
),

ns AS (
    SELECT
        CLI_ID,
        SUM(out_rub) AS ns_sum
    FROM #ns19
    GROUP BY CLI_ID
)

SELECT
    c.CLI_ID,

    CASE WHEN ISNULL(e.exit_td_sum, 0) > 0
        THEN 1 ELSE 0 END AS has_exit_td_flag,

    ISNULL(e.exit_td_count, 0) AS exit_td_count,
    ISNULL(e.exit_td_sum, 0) AS exit_td_sum,

    ISNULL(e.exit_new_money_count, 0)
        AS exit_new_money_count,
    ISNULL(e.exit_new_money_sum, 0)
        AS exit_new_money_sum,

    ISNULL(e.exit_retention_count, 0)
        AS exit_retention_count,
    ISNULL(e.exit_retention_sum, 0)
        AS exit_retention_sum,

    ISNULL(e.exit_crm_count, 0)
        AS exit_crm_count,
    ISNULL(e.exit_crm_sum, 0)
        AS exit_crm_sum,

    CASE WHEN ISNULL(n.ns_sum, 0) > 0
        THEN 1 ELSE 0 END AS has_ns_nonzero_flag,

    ISNULL(n.ns_sum, 0) AS ns_sum

FROM #clients c
LEFT JOIN exits e ON e.CLI_ID = c.CLI_ID
LEFT JOIN ns n ON n.CLI_ID = c.CLI_ID
ORDER BY c.CLI_ID;


/* РЕЗУЛЬТАТ 6. Поклиентные размещения на 10.11 */

;WITH opened AS (
    SELECT
        CLI_ID,

        COUNT(*) AS opened_total_count,
        SUM(out_rub) AS opened_total_sum,

        SUM(CASE WHEN is_15 = 1
            THEN 1 ELSE 0 END) AS opened_15_count,
        SUM(CASE WHEN is_15 = 1
            THEN out_rub ELSE 0 END) AS opened_15_sum,

        SUM(CASE WHEN is_15 = 0
            THEN 1 ELSE 0 END) AS opened_other_count,
        SUM(CASE WHEN is_15 = 0
            THEN out_rub ELSE 0 END) AS opened_other_sum,

        SUM(CASE WHEN category = N'Новые деньги'
            THEN 1 ELSE 0 END) AS opened_new_money_count,
        SUM(CASE WHEN category = N'Новые деньги'
            THEN out_rub ELSE 0 END) AS opened_new_money_sum,

        SUM(CASE WHEN category = N'Удержание'
            THEN 1 ELSE 0 END) AS opened_retention_count,
        SUM(CASE WHEN category = N'Удержание'
            THEN out_rub ELSE 0 END) AS opened_retention_sum,

        SUM(CASE WHEN category = N'CRM'
            THEN 1 ELSE 0 END) AS opened_crm_count,
        SUM(CASE WHEN category = N'CRM'
            THEN out_rub ELSE 0 END) AS opened_crm_sum

    FROM #tagged
    WHERE deal_type = 'OPEN'
    GROUP BY CLI_ID
),

ns AS (
    SELECT
        CLI_ID,
        SUM(out_rub) AS ns_sum
    FROM #snap
    WHERE dt_rep = @dt_to
      AND section_name = N'Накопительный счёт'
    GROUP BY CLI_ID
)

SELECT
    c.CLI_ID,

    ISNULL(o.opened_15_count, 0) AS opened_15_count,
    ISNULL(o.opened_15_sum, 0) AS opened_15_sum,

    ISNULL(o.opened_other_count, 0) AS opened_other_count,
    ISNULL(o.opened_other_sum, 0) AS opened_other_sum,

    ISNULL(o.opened_total_count, 0) AS opened_total_count,
    ISNULL(o.opened_total_sum, 0) AS opened_total_sum,

    ISNULL(o.opened_new_money_count, 0)
        AS opened_new_money_count,
    ISNULL(o.opened_new_money_sum, 0)
        AS opened_new_money_sum,

    ISNULL(o.opened_retention_count, 0)
        AS opened_retention_count,
    ISNULL(o.opened_retention_sum, 0)
        AS opened_retention_sum,

    ISNULL(o.opened_crm_count, 0)
        AS opened_crm_count,
    ISNULL(o.opened_crm_sum, 0)
        AS opened_crm_sum,

    CASE WHEN ISNULL(n.ns_sum, 0) > 0
        THEN 1 ELSE 0 END AS has_ns_nonzero_flag,

    ISNULL(n.ns_sum, 0) AS ns_sum

FROM #clients c
LEFT JOIN opened o ON o.CLI_ID = c.CLI_ID
LEFT JOIN ns n ON n.CLI_ID = c.CLI_ID
ORDER BY c.CLI_ID;


/* РЕЗУЛЬТАТ 7. Контроль сумм */

;WITH checks AS (
    SELECT
        N'Выходящие вклады 19.10' AS check_name,

        (SELECT ISNULL(SUM(out_rub), 0)
         FROM #snap
         WHERE dt_rep = @dt_before
           AND section_name = N'Срочные'
           AND dt_close BETWEEN @dt_promo AND @dt_to)
        AS source_rub,

        (SELECT ISNULL(SUM(out_rub), 0)
         FROM #tagged
         WHERE deal_type = 'EXIT')
        AS detail_rub

    UNION ALL

    SELECT
        N'НС 19.10',

        (SELECT ISNULL(SUM(out_rub), 0)
         FROM #snap
         WHERE dt_rep = @dt_before
           AND section_name = N'Накопительный счёт'),

        (SELECT ISNULL(SUM(out_rub), 0)
         FROM #ns19)

    UNION ALL

    SELECT
        N'Открытия 20.10–10.11',

        (SELECT ISNULL(SUM(out_rub), 0)
         FROM #snap
         WHERE dt_rep = @dt_to
           AND section_name = N'Срочные'
           AND dt_open BETWEEN @dt_promo AND @dt_to),

        (SELECT ISNULL(SUM(out_rub), 0)
         FROM #tagged
         WHERE deal_type = 'OPEN')

    UNION ALL

    SELECT
        N'Весь баланс 19.10',

        (SELECT ISNULL(SUM(out_rub), 0)
         FROM #snap
         WHERE dt_rep = @dt_before),

        (SELECT deposits_rub + ns_rub
         FROM #daily
         WHERE dt_rep = @dt_before)

    UNION ALL

    SELECT
        N'Весь баланс 10.11',

        (SELECT ISNULL(SUM(out_rub), 0)
         FROM #snap
         WHERE dt_rep = @dt_to),

        (SELECT deposits_rub + ns_rub
         FROM #daily
         WHERE dt_rep = @dt_to)
)

SELECT
    check_name,
    source_rub,
    detail_rub,
    detail_rub - source_rub AS difference_rub
FROM checks;
