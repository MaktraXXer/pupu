USE [ALM];
SET NOCOUNT ON;


/* ============================================================
   ПАРАМЕТРЫ
   ============================================================ */

DECLARE @SnapshotDate date = '2026-08-30';

DECLARE @DateFrom date = '2026-09-01';
DECLARE @DateTo   date = '2026-12-31';

/*
    0 = без flat portfolio:
        смотрим только реальные погашения вкладов,
        существовавших на @SnapshotDate

    1 = flat portfolio:
        после погашения вклад считается автоматически
        пролонгированным ровно на тот же срок
*/
DECLARE @FlatPortfolio bit = 1;


/* ============================================================
   TEMP
   ============================================================ */

IF OBJECT_ID('tempdb..#contracts') IS NOT NULL
    DROP TABLE #contracts;

IF OBJECT_ID('tempdb..#maturities') IS NOT NULL
    DROP TABLE #maturities;

IF OBJECT_ID('tempdb..#calendar') IS NOT NULL
    DROP TABLE #calendar;


/* ============================================================
   1. ЧЕСТНЫЙ БАЛАНС НА SNAPSHOT DATE

   Одна строка = один договор.

   Остаток фиксируем на 30.08.2026 и дальше при flat portfolio
   считаем его неизменным.
   ============================================================ */

SELECT
      CAST(t.cli_id AS bigint) AS cli_id
    , CAST(t.con_id AS bigint) AS con_id

    , MIN(CAST(t.dt_open AS date)) AS dt_open
    , MIN(CAST(t.dt_close_plan AS date)) AS dt_close_plan

    , DATEDIFF
      (
          day,
          MIN(CAST(t.dt_open AS date)),
          MIN(CAST(t.dt_close_plan AS date))
      ) AS original_term_days

    , SUM(CAST(t.out_rub AS decimal(38,6))) AS out_rub

INTO #contracts

FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

WHERE
    t.dt_rep = @SnapshotDate

    AND t.section_name = N'Срочные'
    AND t.block_name = N'Привлечение ФЛ'
    AND t.acc_role = N'LIAB'
    AND t.od_flag = 1
    AND t.cur = '810'

    AND t.out_rub IS NOT NULL
    AND t.out_rub > 0

    AND t.dt_open IS NOT NULL
    AND t.dt_close_plan IS NOT NULL

GROUP BY
      t.cli_id
    , t.con_id

HAVING
    DATEDIFF
    (
        day,
        MIN(CAST(t.dt_open AS date)),
        MIN(CAST(t.dt_close_plan AS date))
    ) > 0

OPTION (RECOMPILE);


CREATE UNIQUE CLUSTERED INDEX IX_contracts_con
    ON #contracts (con_id);


/* ============================================================
   2. ДАТЫ ПОГАШЕНИЙ

   Сначала всегда записываем реальное ближайшее погашение.

   При FlatPortfolio = 1:
       следующее погашение =
       предыдущее погашение + original_term_days

   И так продолжаем до конца горизонта, плюс 30 дней,
   потому что для 31.12 нужно смотреть вперёд до 30.01.2027.
   ============================================================ */

CREATE TABLE #maturities
(
      con_id bigint NOT NULL
    , maturity_date date NOT NULL
    , out_rub decimal(38,6) NOT NULL
);


IF @FlatPortfolio = 0
BEGIN

    /* Только реальные погашения */

    INSERT INTO #maturities
    (
          con_id
        , maturity_date
        , out_rub
    )
    SELECT
          con_id
        , dt_close_plan
        , out_rub
    FROM #contracts
    WHERE
        dt_close_plan > @SnapshotDate
        AND dt_close_plan <= DATEADD(day, 30, @DateTo);

END;


ELSE
BEGIN

    /* ========================================================
       FLAT PORTFOLIO

       Рекурсивно строим будущие пролонгации.
       ======================================================== */

    ;WITH maturity_chain AS
    (
        /* Реальный ближайший выход */
        SELECT
              c.con_id
            , c.dt_close_plan AS maturity_date
            , c.original_term_days
            , c.out_rub

        FROM #contracts c

        WHERE
            c.dt_close_plan > @SnapshotDate
            AND c.dt_close_plan <= DATEADD(day, 30, @DateTo)


        UNION ALL


        /* Следующая пролонгация */
        SELECT
              m.con_id

            , DATEADD(
                  day,
                  m.original_term_days,
                  m.maturity_date
              ) AS maturity_date

            , m.original_term_days
            , m.out_rub

        FROM maturity_chain m

        WHERE
            DATEADD(
                day,
                m.original_term_days,
                m.maturity_date
            ) <= DATEADD(day, 30, @DateTo)
    )

    INSERT INTO #maturities
    (
          con_id
        , maturity_date
        , out_rub
    )

    SELECT
          con_id
        , maturity_date
        , out_rub

    FROM maturity_chain

    OPTION (MAXRECURSION 1000);

END;


CREATE INDEX IX_maturities_date
    ON #maturities (maturity_date);


/* ============================================================
   3. КАЛЕНДАРЬ НАБЛЮДЕНИЙ

   01.09.2026 ... 31.12.2026
   ============================================================ */

;WITH calendar AS
(
    SELECT
        @DateFrom AS dt_rep

    UNION ALL

    SELECT
        DATEADD(day, 1, dt_rep)

    FROM calendar

    WHERE dt_rep < @DateTo
)

SELECT
    dt_rep
INTO #calendar
FROM calendar

OPTION (MAXRECURSION 1000);


CREATE UNIQUE CLUSTERED INDEX IX_calendar
    ON #calendar (dt_rep);


/* ============================================================
   4. ИТОГ

   Для каждой даты t:

       exit_from = t + 1
       exit_to   = t + 30

   Например:

       dt_rep = 01.09
       считаем погашения 02.09–01.10 включительно.

       dt_rep = 02.09
       считаем погашения 03.09–02.10 включительно.

   Это ровно 30 календарных дней.
   ============================================================ */

SELECT
      c.dt_rep

    , DATEADD(day, 1, c.dt_rep)
        AS exit_from

    , DATEADD(day, 30, c.dt_rep)
        AS exit_to

    , ISNULL(
          SUM(m.out_rub),
          0
      ) AS deposits_exit_next_30d

FROM #calendar c

LEFT JOIN #maturities m
    ON m.maturity_date >= DATEADD(day, 1, c.dt_rep)
   AND m.maturity_date <= DATEADD(day, 30, c.dt_rep)

GROUP BY
    c.dt_rep

ORDER BY
    c.dt_rep;
