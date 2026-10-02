USE [ALM];
SET NOCOUNT ON;
SET DEADLOCK_PRIORITY LOW;


/* ============================================================
   ПАРАМЕТРЫ
   ============================================================ */

DECLARE @HistoryFrom  date = '2026-07-01';
DECLARE @BaseDate     date = '2026-09-22';
DECLARE @AnalysisDate date = '2026-10-01';

DECLARE @OpenFrom     date = '2026-09-22';
DECLARE @OpenTo       date = '2026-10-01';

/*
    7 = читать историю недельными кусками.
    Если БД тяжело -> поставить 3.
*/
DECLARE @ChunkDays int = 7;


/* ============================================================
   CLEANUP
   ============================================================ */

DROP TABLE IF EXISTS #pk7_openings;
DROP TABLE IF EXISTS #pk7_clients;
DROP TABLE IF EXISTS #history;
DROP TABLE IF EXISTS #bal_2209;
DROP TABLE IF EXISTS #bal_0110;
DROP TABLE IF EXISTS #relevant_con_ids;
DROP TABLE IF EXISTS #attr_latest;
DROP TABLE IF EXISTS #exit_contracts;



/* ============================================================
   1. ВКЛАДЫ Пк7, ОТКРЫТЫЕ 22.09 - 01.10

   Сначала берём последнюю запись по каждому CON_ID.
   ============================================================ */

;WITH a AS
(
    SELECT
          TRY_CAST(x.CON_ID AS bigint) AS con_id
        , TRY_CAST(x.CLI_ID AS bigint) AS cli_id

        , x.DT_OPEN_FACT
        , x.DT_CLOSE_PLAN
        , x.DT_CLOSE_FACT

        , x.DEPOSIT_ADD_CONDITIONS
        , x.PROMO_CODE
        , x.PROMO_GROUP
        , x.START_DEPOSIT

        , ISNULL(TRY_CAST(x.[Пк7] AS int),0) AS pk7

        , ROW_NUMBER() OVER
          (
              PARTITION BY x.CON_ID
              ORDER BY
                    x.DT_UPDATE DESC
                  , x.loaddate DESC
          ) AS rn

    FROM [ALM].[ehd].[attr_DepoFLConditions] x WITH (NOLOCK)

    WHERE
            x.DT_OPEN_FACT >= @OpenFrom
        AND x.DT_OPEN_FACT < DATEADD(day,1,@OpenTo)
)

SELECT
      con_id
    , cli_id

    , CAST(DT_OPEN_FACT AS date)  AS dt_open_fact
    , CAST(DT_CLOSE_PLAN AS date) AS dt_close_plan
    , CAST(DT_CLOSE_FACT AS date) AS dt_close_fact

    , DEPOSIT_ADD_CONDITIONS
    , PROMO_CODE
    , PROMO_GROUP
    , START_DEPOSIT

INTO #pk7_openings

FROM a

WHERE
        rn = 1
    AND pk7 = 1
    AND cli_id IS NOT NULL
    AND con_id IS NOT NULL;


CREATE UNIQUE CLUSTERED INDEX IX_pk7_openings_con
    ON #pk7_openings(con_id);

CREATE INDEX IX_pk7_openings_cli
    ON #pk7_openings(cli_id);



/* ============================================================
   2. КОГОРТА КЛИЕНТОВ

   ВАЖНО:
   создаём cli_id ТОГО ЖЕ ТИПА,
   что cli_id во VW_balance_rest_all.

   Поэтому дальше не нужен CAST(t.cli_id...)
   на огромной view.
   ============================================================ */

SELECT TOP (0)
    t.cli_id

INTO #pk7_clients

FROM [ALM].[ALM].[VW_balance_rest_all] t;


INSERT INTO #pk7_clients
(
    cli_id
)
SELECT DISTINCT
    p.cli_id

FROM #pk7_openings p;


CREATE UNIQUE CLUSTERED INDEX IX_pk7_clients_cli
    ON #pk7_clients(cli_id);



/* ============================================================
   РЕЗУЛЬТАТ 0
   КТО ВОШЁЛ В КОГОРТУ Пк7
   ============================================================ */

SELECT
      cli_id
    , con_id
    , dt_open_fact
    , dt_close_plan
    , START_DEPOSIT
    , DEPOSIT_ADD_CONDITIONS
    , PROMO_CODE
    , PROMO_GROUP

FROM #pk7_openings

ORDER BY
      cli_id
    , con_id;



/* ============================================================
   3. ИСТОРИЯ БАЛАНСА

   ВМЕСТО ОДНОГО ЗАПРОСА ЗА 84 ДНЯ
   ЧИТАЕМ КУСКАМИ ПО @ChunkDays.

   В историю сохраняется только:
   дата / вклады / НС / всего

   То есть не раздуваем temp таблицу клиентскими строками.
   ============================================================ */

CREATE TABLE #history
(
      dt_rep    date NOT NULL
    , td_sum    decimal(38,2) NOT NULL
    , ns_sum    decimal(38,2) NOT NULL
    , total_sum decimal(38,2) NOT NULL
);


DECLARE @ChunkFrom date = @HistoryFrom;
DECLARE @ChunkTo   date;


WHILE @ChunkFrom <= @BaseDate
BEGIN

    SET @ChunkTo =
        DATEADD(day,@ChunkDays,@ChunkFrom);

    IF @ChunkTo > DATEADD(day,1,@BaseDate)
        SET @ChunkTo = DATEADD(day,1,@BaseDate);


    INSERT INTO #history
    (
          dt_rep
        , td_sum
        , ns_sum
        , total_sum
    )

    SELECT
          t.dt_rep

        , SUM(
            CASE
                WHEN t.section_name = N'Срочные'
                    THEN t.out_rub
                ELSE 0
            END
          ) AS td_sum

        , SUM(
            CASE
                WHEN t.section_name = N'Накопительный счёт'
                    THEN t.out_rub
                ELSE 0
            END
          ) AS ns_sum

        , SUM(t.out_rub) AS total_sum

    FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)

    INNER JOIN #pk7_clients c
        ON c.cli_id = t.cli_id

    WHERE
            t.dt_rep >= @ChunkFrom
        AND t.dt_rep <  @ChunkTo

        AND t.section_name IN
        (
            N'Срочные',
            N'Накопительный счёт'
        )

        AND t.block_name = N'Привлечение ФЛ'
        AND t.acc_role   = N'LIAB'
        AND t.od_flag    = 1
        AND t.cur        = '810'

        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0

    GROUP BY
        t.dt_rep

    OPTION
    (
        RECOMPILE,
        MAXDOP 2
    );


    SET @ChunkFrom = @ChunkTo;


    /*
        Небольшая пауза между тяжёлыми чтениями.
        Можно убрать ночью.
    */
    IF @ChunkFrom <= @BaseDate
        WAITFOR DELAY '00:00:01';

END;



/* ============================================================
   РЕЗУЛЬТАТ 1
   ДИНАМИКА КОГОРТЫ
   ============================================================ */

SELECT
      dt_rep
    , td_sum
    , ns_sum
    , total_sum

FROM #history

ORDER BY dt_rep;



/* ============================================================
   4. SNAPSHOT 22.09

   Один день -> намного легче истории.
   ============================================================ */

SELECT
      t.cli_id
    , t.con_id

    , CAST(t.dt_open AS date)       AS dt_open
    , CAST(t.dt_close_plan AS date) AS dt_close_plan

    , t.section_name
    , t.PROD_NAME_res
    , t.TSEGMENTNAME

    , CAST(t.out_rub AS decimal(38,2)) AS out_rub

    , t.rate_con
    , t.termdays

INTO #bal_2209

FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)

INNER JOIN #pk7_clients c
    ON c.cli_id = t.cli_id

WHERE
        t.dt_rep = @BaseDate

    AND t.section_name IN
    (
        N'Срочные',
        N'Накопительный счёт'
    )

    AND t.block_name = N'Привлечение ФЛ'
    AND t.acc_role   = N'LIAB'
    AND t.od_flag    = 1
    AND t.cur        = '810'

    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0

OPTION
(
    RECOMPILE,
    MAXDOP 2
);


CREATE INDEX IX_bal_2209_cli
    ON #bal_2209(cli_id);

CREATE INDEX IX_bal_2209_con
    ON #bal_2209(con_id);



/* ============================================================
   5. SNAPSHOT 01.10
   ============================================================ */

SELECT
      t.cli_id
    , t.con_id

    , CAST(t.dt_open AS date)       AS dt_open
    , CAST(t.dt_close_plan AS date) AS dt_close_plan

    , t.section_name
    , t.PROD_NAME_res
    , t.TSEGMENTNAME

    , CAST(t.out_rub AS decimal(38,2)) AS out_rub

    , t.rate_con
    , t.termdays

INTO #bal_0110

FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)

INNER JOIN #pk7_clients c
    ON c.cli_id = t.cli_id

WHERE
        t.dt_rep = @AnalysisDate

    AND t.section_name IN
    (
        N'Срочные',
        N'Накопительный счёт'
    )

    AND t.block_name = N'Привлечение ФЛ'
    AND t.acc_role   = N'LIAB'
    AND t.od_flag    = 1
    AND t.cur        = '810'

    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0

OPTION
(
    RECOMPILE,
    MAXDOP 2
);


CREATE INDEX IX_bal_0110_cli
    ON #bal_0110(cli_id);

CREATE INDEX IX_bal_0110_con
    ON #bal_0110(con_id);



/* ============================================================
   6. НУЖНЫЕ CON_ID
   ============================================================ */

SELECT DISTINCT
    con_id

INTO #relevant_con_ids

FROM
(
    SELECT con_id
    FROM #bal_2209
    WHERE con_id IS NOT NULL

    UNION ALL

    SELECT con_id
    FROM #bal_0110
    WHERE con_id IS NOT NULL

    UNION ALL

    SELECT con_id
    FROM #pk7_openings
    WHERE con_id IS NOT NULL
) x;


CREATE UNIQUE CLUSTERED INDEX IX_relevant_con_ids
    ON #relevant_con_ids(con_id);



/* ============================================================
   7. ПОСЛЕДНИЕ ПРИЗНАКИ НАДБАВОК
   ============================================================ */

;WITH a AS
(
    SELECT
          TRY_CAST(x.CON_ID AS bigint) AS con_id

        , x.DEPOSIT_ADD_CONDITIONS
        , x.PROMO_CODE
        , x.PROMO_GROUP
        , x.START_DEPOSIT

        , ISNULL(TRY_CAST(x.[Зрп] AS int),0) AS [Зрп]
        , ISNULL(TRY_CAST(x.[Пнс] AS int),0) AS [Пнс]
        , ISNULL(TRY_CAST(x.[Нов] AS int),0) AS [Нов]

        , ISNULL(TRY_CAST(x.[НДП] AS int),0) AS [НДП]
        , ISNULL(TRY_CAST(x.[НДМ] AS int),0) AS [НДМ]

        , ISNULL(TRY_CAST(x.[Мпл] AS int),0) AS [Мпл]

        , ISNULL(TRY_CAST(x.[Прл] AS int),0) AS [Прл]
        , ISNULL(TRY_CAST(x.[Пр2] AS int),0) AS [Пр2]
        , ISNULL(TRY_CAST(x.[Пр3] AS int),0) AS [Пр3]

        , ISNULL(TRY_CAST(x.[Пк1] AS int),0) AS [Пк1]
        , ISNULL(TRY_CAST(x.[Пк2] AS int),0) AS [Пк2]
        , ISNULL(TRY_CAST(x.[Пк3] AS int),0) AS [Пк3]
        , ISNULL(TRY_CAST(x.[Пк4] AS int),0) AS [Пк4]
        , ISNULL(TRY_CAST(x.[Пк5] AS int),0) AS [Пк5]
        , ISNULL(TRY_CAST(x.[Пк6] AS int),0) AS [Пк6]
        , ISNULL(TRY_CAST(x.[Пк7] AS int),0) AS [Пк7]

        , ISNULL(TRY_CAST(x.[От1] AS int),0) AS [От1]
        , ISNULL(TRY_CAST(x.[От2] AS int),0) AS [От2]
        , ISNULL(TRY_CAST(x.[От3] AS int),0) AS [От3]

        , ISNULL(TRY_CAST(x.[ДБО] AS int),0) AS [ДБО]
        , ISNULL(TRY_CAST(x.[Лмт] AS int),0) AS [Лмт]
        , ISNULL(TRY_CAST(x.[Прм] AS int),0) AS [Прм]

        , ROW_NUMBER() OVER
          (
              PARTITION BY x.CON_ID
              ORDER BY
                    x.DT_UPDATE DESC
                  , x.loaddate DESC
          ) AS rn

    FROM [ALM].[ehd].[attr_DepoFLConditions] x WITH (NOLOCK)

    INNER JOIN #relevant_con_ids r
        ON r.con_id = TRY_CAST(x.CON_ID AS bigint)
)

SELECT
      con_id

    , DEPOSIT_ADD_CONDITIONS
    , PROMO_CODE
    , PROMO_GROUP
    , START_DEPOSIT

    , [Зрп]
    , [Пнс]
    , [Нов]

    , [НДП]
    , [НДМ]

    , [Мпл]

    , [Прл]
    , [Пр2]
    , [Пр3]

    , [Пк1]
    , [Пк2]
    , [Пк3]
    , [Пк4]
    , [Пк5]
    , [Пк6]
    , [Пк7]

    , [От1]
    , [От2]
    , [От3]

    , [ДБО]
    , [Лмт]
    , [Прм]

INTO #attr_latest

FROM a

WHERE rn = 1;


CREATE UNIQUE CLUSTERED INDEX IX_attr_latest
    ON #attr_latest(con_id);



/* ============================================================
   8. ВКЛАДЫ К ВЫХОДУ НА 22.09
   ============================================================ */

SELECT
      b.cli_id
    , b.con_id

    , b.dt_open
    , b.dt_close_plan

    , DATEDIFF(day,b.dt_open,b.dt_close_plan)
        AS term_days_calc

    , b.termdays

    , b.PROD_NAME_res
    , b.TSEGMENTNAME

    , b.out_rub
    , b.rate_con

    , a.START_DEPOSIT
    , a.DEPOSIT_ADD_CONDITIONS
    , a.PROMO_CODE
    , a.PROMO_GROUP

    , a.[Зрп]
    , a.[Пнс]
    , a.[Нов]

    , a.[НДП]
    , a.[НДМ]

    , a.[Мпл]

    , a.[Прл]
    , a.[Пр2]
    , a.[Пр3]

    , a.[Пк1]
    , a.[Пк2]
    , a.[Пк3]
    , a.[Пк4]
    , a.[Пк5]
    , a.[Пк6]
    , a.[Пк7]

    , a.[От1]
    , a.[От2]
    , a.[От3]

    , a.[ДБО]
    , a.[Лмт]
    , a.[Прм]

INTO #exit_contracts

FROM #bal_2209 b

LEFT JOIN #attr_latest a
    ON a.con_id = b.con_id

WHERE
        b.section_name = N'Срочные'

    AND b.dt_close_plan >= @BaseDate
    AND b.dt_close_plan <= @AnalysisDate;



/* ============================================================
   РЕЗУЛЬТАТ 2
   ПОКОНТРАКТНЫЕ ВКЛАДЫ К ВЫХОДУ
   ============================================================ */

SELECT *
FROM #exit_contracts

ORDER BY
      cli_id
    , dt_close_plan
    , con_id;



/* ============================================================
   РЕЗУЛЬТАТ 3
   КЛИЕНТ НА 22.09

   - есть вклад к выходу
   - объём
   - количество
   - остаток НС
   ============================================================ */

;WITH exit_agg AS
(
    SELECT
          cli_id
        , COUNT(DISTINCT con_id) AS exit_td_count
        , SUM(out_rub)           AS exit_td_sum

    FROM #exit_contracts

    GROUP BY cli_id
),

ns AS
(
    SELECT
          cli_id
        , SUM(out_rub) AS ns_sum

    FROM #bal_2209

    WHERE
        section_name = N'Накопительный счёт'

    GROUP BY cli_id
)

SELECT
      c.cli_id

    , CASE
          WHEN e.cli_id IS NOT NULL
              THEN 1
          ELSE 0
      END AS has_exit_td_flag

    , ISNULL(e.exit_td_count,0)
        AS exit_td_count

    , ISNULL(e.exit_td_sum,0)
        AS exit_td_sum

    , CASE
          WHEN ISNULL(n.ns_sum,0) > 0
              THEN 1
          ELSE 0
      END AS has_ns_nonzero_flag

    , ISNULL(n.ns_sum,0)
        AS ns_sum

FROM #pk7_clients c

LEFT JOIN exit_agg e
    ON e.cli_id = c.cli_id

LEFT JOIN ns n
    ON n.cli_id = c.cli_id

ORDER BY
    c.cli_id;



/* ============================================================
   РЕЗУЛЬТАТ 4
   ЧТО КЛИЕНТ ИМЕЕТ НА 01.10

   Вклады, открытые 22.09-01.10:
   - Пк7
   - остальные
   - НС на 01.10
   ============================================================ */

;WITH opened_by_con AS
(
    SELECT
          b.cli_id
        , b.con_id

        , MAX(
            CASE
                WHEN ISNULL(a.[Пк7],0) = 1
                    THEN 1
                ELSE 0
            END
          ) AS is_pk7

        , SUM(b.out_rub) AS out_rub

    FROM #bal_0110 b

    LEFT JOIN #attr_latest a
        ON a.con_id = b.con_id

    WHERE
            b.section_name = N'Срочные'

        AND b.dt_open >= @OpenFrom
        AND b.dt_open <= @OpenTo

    GROUP BY
          b.cli_id
        , b.con_id
),

opened_client AS
(
    SELECT
          cli_id

        , SUM(
            CASE
                WHEN is_pk7 = 1 THEN 1
                ELSE 0
            END
          ) AS opened_pk7_count

        , SUM(
            CASE
                WHEN is_pk7 = 1 THEN out_rub
                ELSE 0
            END
          ) AS opened_pk7_sum

        , SUM(
            CASE
                WHEN is_pk7 = 0 THEN 1
                ELSE 0
            END
          ) AS opened_other_count

        , SUM(
            CASE
                WHEN is_pk7 = 0 THEN out_rub
                ELSE 0
            END
          ) AS opened_other_sum

        , COUNT(*) AS opened_total_count
        , SUM(out_rub) AS opened_total_sum

    FROM opened_by_con

    GROUP BY cli_id
),

ns AS
(
    SELECT
          cli_id
        , SUM(out_rub) AS ns_0110_sum

    FROM #bal_0110

    WHERE
        section_name = N'Накопительный счёт'

    GROUP BY cli_id
)

SELECT
      c.cli_id

    , ISNULL(o.opened_pk7_count,0)
        AS opened_pk7_count

    , ISNULL(o.opened_pk7_sum,0)
        AS opened_pk7_sum

    , ISNULL(o.opened_other_count,0)
        AS opened_other_count

    , ISNULL(o.opened_other_sum,0)
        AS opened_other_sum

    , ISNULL(o.opened_total_count,0)
        AS opened_total_count

    , ISNULL(o.opened_total_sum,0)
        AS opened_total_sum

    , ISNULL(n.ns_0110_sum,0)
        AS ns_0110_sum

FROM #pk7_clients c

LEFT JOIN opened_client o
    ON o.cli_id = c.cli_id

LEFT JOIN ns n
    ON n.cli_id = c.cli_id

ORDER BY
    c.cli_id;
