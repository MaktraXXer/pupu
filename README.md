
USE ALM;
SET NOCOUNT ON;

DECLARE @dt_from date = '2026-09-01';
DECLARE @dt_before date = '2026-10-19';
DECLARE @dt_promo date = '2026-10-20';
DECLARE @dt_to date = '2026-11-10';

DROP TABLE IF EXISTS #balance10;
DROP TABLE IF EXISTS #clients15;
DROP TABLE IF EXISTS #history;
DROP TABLE IF EXISTS #daily;
DROP TABLE IF EXISTS #exit_base;
DROP TABLE IF EXISTS #ns19;
DROP TABLE IF EXISTS #opened_base;
DROP TABLE IF EXISTS #attr_ids;
DROP TABLE IF EXISTS #attrs;
DROP TABLE IF EXISTS #exit;
DROP TABLE IF EXISTS #opened;


/* 1. Баланс на 10 ноября.
   Сначала берем вклады и НС, чтобы определить клиентов,
   открывших двухмесячные вклады под 15%. */

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
    CAST(t.OUT_RUB AS decimal(28,2)) AS out_rub

INTO #balance10

FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

WHERE t.dt_rep = @dt_to
  AND t.block_name = N'Привлечение ФЛ'
  AND t.section_name IN (
      N'Срочные',
      N'Накопительный счёт'
  )
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.OUT_RUB IS NOT NULL
  AND t.OUT_RUB >= 0

  AND (
      t.section_name <> N'Накопительный счёт'
      OR t.acc_role = N'LIAB'
  );

CREATE INDEX IX_balance10
ON #balance10 (CLI_ID, section_name);


/* 2. Целевая когорта клиентов.
   Достаточно одного подходящего вклада на 10 ноября. */

SELECT DISTINCT
    CLI_ID
INTO #clients15

FROM #balance10

WHERE CLI_ID IS NOT NULL
  AND section_name = N'Срочные'
  AND out_rub > 0

  AND dt_open BETWEEN @dt_promo AND @dt_to
  AND termdays BETWEEN 45 AND 70
  AND ABS(rate_con - 0.15) <= 0.0005
  AND ISNULL(conv, 'AT_THE_END') = 'AT_THE_END';

CREATE UNIQUE CLUSTERED INDEX IX_clients15
ON #clients15 (CLI_ID);


/* 3. История остатков только целевых клиентов.
   Весь последующий анализ строится из этой таблицы.

   10 ноября берем из уже загруженного #balance10,
   поэтому эта дата не читается второй раз. */

SELECT
    b.*
INTO #history

FROM #balance10 b

INNER JOIN #clients15 c
    ON c.CLI_ID = b.CLI_ID

UNION ALL

SELECT
    CAST(t.dt_rep AS date),
    t.CLI_ID,
    t.CON_ID,
    t.TSEGMENTNAME,
    t.PROD_NAME_res,
    t.section_name,
    CAST(t.dt_open AS date),
    CAST(t.dt_close AS date),
    t.rate_con,
    t.conv,
    t.termdays,
    CAST(t.OUT_RUB AS decimal(28,2))

FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

INNER JOIN #clients15 c
    ON c.CLI_ID = t.CLI_ID

WHERE t.dt_rep >= @dt_from
  AND t.dt_rep < @dt_to

  AND t.block_name = N'Привлечение ФЛ'
  AND t.section_name IN (
      N'Срочные',
      N'Накопительный счёт'
  )
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.OUT_RUB IS NOT NULL
  AND t.OUT_RUB >= 0

  AND (
      t.section_name <> N'Накопительный счёт'
      OR t.acc_role = N'LIAB'
  )

OPTION (MAXDOP 2, RECOMPILE);

CREATE CLUSTERED INDEX IX_history
ON #history (dt_rep, CLI_ID);


/* 4. Подневные остатки вкладов и НС.
   Добавляем календарь, чтобы не пропускать даты
   с нулевыми остатками. */

;WITH calendar AS (
    SELECT @dt_from AS dt_rep

    UNION ALL

    SELECT DATEADD(day, 1, dt_rep)
    FROM calendar
    WHERE dt_rep < @dt_to
)

SELECT
    c.dt_rep,

    SUM(CASE
        WHEN h.section_name = N'Срочные'
            THEN h.out_rub
        ELSE 0
    END) AS deposits_rub,

    SUM(CASE
        WHEN h.section_name = N'Накопительный счёт'
            THEN h.out_rub
        ELSE 0
    END) AS ns_rub

INTO #daily

FROM calendar c

LEFT JOIN #history h
    ON h.dt_rep = c.dt_rep

GROUP BY c.dt_rep

OPTION (MAXRECURSION 0);

CREATE UNIQUE CLUSTERED INDEX IX_daily
ON #daily (dt_rep);


/* 5. Вклады на 19 октября, плановое закрытие
   которых приходится на 20.10–10.11.

   Суммируем все балансовые строки одного договора.
   Никакие договоры из-за отсутствия атрибутов
   на этом этапе не исключаем. */

SELECT
    CLI_ID,
    CON_ID,

    MAX(TSEGMENTNAME) AS TSEGMENTNAME,
    MAX(PROD_NAME_res) AS PROD_NAME_res,
    MAX(rate_con) AS rate_con,
    MAX(conv) AS conv,
    MAX(termdays) AS termdays,
    MAX(dt_open) AS dt_open,
    MAX(dt_close) AS dt_close,

    SUM(out_rub) AS out_rub

INTO #exit_base

FROM #history

WHERE dt_rep = @dt_before
  AND section_name = N'Срочные'
  AND dt_close BETWEEN @dt_promo AND @dt_to

GROUP BY CLI_ID, CON_ID

HAVING SUM(out_rub) > 0;

CREATE INDEX IX_exit_base
ON #exit_base (CLI_ID, CON_ID);


/* 6. Все НС на 19 октября.
   Без ограничений по открытию или закрытию. */

SELECT
    CLI_ID,
    CON_ID,

    MAX(TSEGMENTNAME) AS TSEGMENTNAME,
    MAX(PROD_NAME_res) AS PROD_NAME_res,
    MAX(rate_con) AS rate_con,

    SUM(out_rub) AS out_rub

INTO #ns19

FROM #history

WHERE dt_rep = @dt_before
  AND section_name = N'Накопительный счёт'

GROUP BY CLI_ID, CON_ID;

CREATE INDEX IX_ns19
ON #ns19 (CLI_ID);


/* 7. Все срочные вклады целевых клиентов на 10 ноября,
   открытые с 20 октября.

   Отдельно отмечаем подходящие вклады под 15%. */

SELECT
    CLI_ID,
    CON_ID,

    MAX(TSEGMENTNAME) AS TSEGMENTNAME,
    MAX(PROD_NAME_res) AS PROD_NAME_res,
    MAX(rate_con) AS rate_con,
    MAX(conv) AS conv,
    MAX(termdays) AS termdays,
    MAX(dt_open) AS dt_open,
    MAX(dt_close) AS dt_close,

    MAX(CASE
        WHEN termdays BETWEEN 45 AND 70
         AND ABS(rate_con - 0.15) <= 0.0005
         AND ISNULL(conv, 'AT_THE_END') = 'AT_THE_END'
            THEN 1
        ELSE 0
    END) AS is_15,

    SUM(out_rub) AS out_rub

INTO #opened_base

FROM #history

WHERE dt_rep = @dt_to
  AND section_name = N'Срочные'
  AND dt_open BETWEEN @dt_promo AND @dt_to

GROUP BY CLI_ID, CON_ID

HAVING SUM(out_rub) > 0;

CREATE INDEX IX_opened_base
ON #opened_base (CLI_ID, CON_ID);


/* 8. Получаем последние надбавки для договоров
   из двух посделочных выборок.

   Отсутствие записи в таблице надбавок
   не исключает договор из анализа. */

SELECT DISTINCT
    CON_ID

INTO #attr_ids

FROM (
    SELECT CON_ID FROM #exit_base
    UNION ALL
    SELECT CON_ID FROM #opened_base
) x

WHERE CON_ID IS NOT NULL;

CREATE UNIQUE CLUSTERED INDEX IX_attr_ids
ON #attr_ids (CON_ID);


SELECT
    i.CON_ID,

    ISNULL(TRY_CAST(a.[Нов] AS int), 0) AS nov,
    ISNULL(TRY_CAST(a.[НДП] AS int), 0) AS ndp,
    ISNULL(TRY_CAST(a.[НДМ] AS int), 0) AS ndm,

    ISNULL(TRY_CAST(a.[ПК3] AS int), 0) AS pk3,
    ISNULL(TRY_CAST(a.[ОТ1] AS int), 0) AS ot1,

    ISNULL(TRY_CAST(a.[КП3] AS int), 0) AS kp3,
    ISNULL(TRY_CAST(a.[ПК6] AS int), 0) AS pk6,
    ISNULL(TRY_CAST(a.[ПК7] AS int), 0) AS pk7

INTO #attrs

FROM #attr_ids i

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

    WHERE a.CON_ID = CONVERT(varchar(30), i.CON_ID)

    ORDER BY
        a.DT_UPDATE DESC,
        a.loaddate DESC
) a;

CREATE UNIQUE CLUSTERED INDEX IX_attrs
ON #attrs (CON_ID);


/* 9. Категоризируем выходящие вклады.
   Приоритет: новые деньги -> удержание -> CRM.
   Один договор попадает только в одну категорию. */

SELECT
    e.*,

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

INTO #exit

FROM #exit_base e

LEFT JOIN #attrs a
    ON a.CON_ID = e.CON_ID;

CREATE INDEX IX_exit
ON #exit (CLI_ID);


/* 10. Аналогично классифицируем новые открытия. */

SELECT
    o.*,

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

INTO #opened

FROM #opened_base o

LEFT JOIN #attrs a
    ON a.CON_ID = o.CON_ID;

CREATE INDEX IX_opened
ON #opened (CLI_ID);


/* ============================================================
   РЕЗУЛЬТАТ А1.
   Подневные остатки 01.09–19.10.
   Только клиенты, открывшие 15% к 10 ноября.
   ============================================================ */

SELECT
    dt_rep,
    deposits_rub,
    ns_rub,
    deposits_rub + ns_rub AS total_rub

FROM #daily

WHERE dt_rep BETWEEN @dt_from AND @dt_before

ORDER BY dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ А2.
   Те же остатки за весь период до 10 ноября.
   ============================================================ */

SELECT
    dt_rep,
    deposits_rub,
    ns_rub,
    deposits_rub + ns_rub AS total_rub

FROM #daily

ORDER BY dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ Б.
   Посделочно: вклады к выходу на 19 октября.
   Остатки и все запрошенные надбавки.
   ============================================================ */

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

    category AS [Категория выхода]

FROM #exit

ORDER BY dt_close, CLI_ID, CON_ID;


/* ============================================================
   РЕЗУЛЬТАТ В.
   Все накопительные счета на 19 октября.
   ============================================================ */

SELECT
    CLI_ID,
    CON_ID,
    TSEGMENTNAME,
    PROD_NAME_res,
    rate_con,
    out_rub

FROM #ns19

ORDER BY CLI_ID, CON_ID;


/* ============================================================
   РЕЗУЛЬТАТ Г.
   Поклиентная витрина на 19 октября.

   Выходящие вклады — с закрытием 20.10–10.11.
   НС — весь остаток на 19.10.
   ============================================================ */

;WITH exit_agg AS (
    SELECT
        CLI_ID,

        COUNT(CON_ID) AS exit_td_count,
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
            THEN out_rub ELSE 0 END) AS exit_crm_sum,

        SUM(CASE WHEN category = N'Остальные'
            THEN out_rub ELSE 0 END) AS exit_other_sum

    FROM #exit
    GROUP BY CLI_ID
),

ns_agg AS (
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

    ISNULL(e.exit_other_sum, 0)
        AS exit_other_sum,

    CASE WHEN ISNULL(n.ns_sum, 0) > 0
        THEN 1 ELSE 0 END AS has_ns_nonzero_flag,

    ISNULL(n.ns_sum, 0) AS ns_sum

FROM #clients15 c

LEFT JOIN exit_agg e
    ON e.CLI_ID = c.CLI_ID

LEFT JOIN ns_agg n
    ON n.CLI_ID = c.CLI_ID

ORDER BY c.CLI_ID;


/* ============================================================
   РЕЗУЛЬТАТ Д.
   Поклиентная витрина на 10 ноября.

   15% и другие вклады — открытые с 20 октября.
   Новые деньги / удержание / CRM — по надбавкам.
   НС — общий остаток на 10 ноября.
   ============================================================ */

;WITH opened_agg AS (
    SELECT
        CLI_ID,

        COUNT(CON_ID) AS opened_total_count,
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

    FROM #opened
    GROUP BY CLI_ID
),

ns_agg AS (
    SELECT
        CLI_ID,
        SUM(out_rub) AS ns_sum
    FROM #history
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

FROM #clients15 c

LEFT JOIN opened_agg o
    ON o.CLI_ID = c.CLI_ID

LEFT JOIN ns_agg n
    ON n.CLI_ID = c.CLI_ID

ORDER BY c.CLI_ID;


/* ============================================================
   РЕЗУЛЬТАТ Е.
   Контроль сохранения объёмов.

   Сравниваем посделочные суммы с исходными строками
   #history, использованными для подневных агрегатов.
   ============================================================ */

;WITH checks AS (
    SELECT
        N'Выходящие вклады на 19.10' AS check_name,

        (
            SELECT ISNULL(SUM(out_rub), 0)
            FROM #history
            WHERE dt_rep = @dt_before
              AND section_name = N'Срочные'
              AND dt_close BETWEEN @dt_promo AND @dt_to
              AND out_rub > 0
        ) AS source_rub,

        (
            SELECT ISNULL(SUM(out_rub), 0)
            FROM #exit
        ) AS detail_rub

    UNION ALL

    SELECT
        N'НС на 19.10',

        (
            SELECT ISNULL(SUM(out_rub), 0)
            FROM #history
            WHERE dt_rep = @dt_before
              AND section_name = N'Накопительный счёт'
        ),

        (
            SELECT ISNULL(SUM(out_rub), 0)
            FROM #ns19
        )

    UNION ALL

    SELECT
        N'Открытые вклады на 10.11',

        (
            SELECT ISNULL(SUM(out_rub), 0)
            FROM #history
            WHERE dt_rep = @dt_to
              AND section_name = N'Срочные'
              AND dt_open BETWEEN @dt_promo AND @dt_to
              AND out_rub > 0
        ),

        (
            SELECT ISNULL(SUM(out_rub), 0)
            FROM #opened
        )
)

SELECT
    check_name AS [Проверка],
    source_rub AS [Баланс, руб],
    detail_rub AS [Посделочная выгрузка, руб],
    detail_rub - source_rub AS [Разница, руб]

FROM checks;
