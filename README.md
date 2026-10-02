USE [ALM];
SET NOCOUNT ON;


/* ============================================================
   ПАРАМЕТРЫ
   ============================================================ */

DECLARE @OpenFrom      date = '2026-09-22';
DECLARE @OpenTo        date = '2026-10-01';

DECLARE @HistoryFrom   date = '2026-07-01';
DECLARE @BaseDate      date = '2026-09-22';
DECLARE @AnalysisDate  date = '2026-10-01';


/* ============================================================
   ЧИСТИМ TEMP
   ============================================================ */

DROP TABLE IF EXISTS #pk7_clients;
DROP TABLE IF EXISTS #pk7_openings;
DROP TABLE IF EXISTS #bal_2209;
DROP TABLE IF EXISTS #bal_0110;
DROP TABLE IF EXISTS #relevant_con_ids;
DROP TABLE IF EXISTS #attr_latest;
DROP TABLE IF EXISTS #exit_contracts;


/* ============================================================
   1. КОГОРТА:
      клиенты, открывшие вклад с Пк7
      с 22.09.2026 по 01.10.2026 включительно
   ============================================================ */

SELECT DISTINCT
      TRY_CAST(a.CLI_ID AS bigint)      AS cli_id
    , TRY_CAST(a.CON_ID AS bigint)      AS con_id
    , CAST(a.DT_OPEN_FACT AS date)      AS dt_open_fact
    , CAST(a.DT_CLOSE_PLAN AS date)     AS dt_close_plan
    , CAST(a.DT_CLOSE_FACT AS date)     AS dt_close_fact

    , a.DEPOSIT_ADD_CONDITIONS
    , a.PROMO_CODE
    , a.PROMO_GROUP
    , a.START_DEPOSIT

INTO #pk7_openings

FROM [ALM].[ehd].[attr_DepoFLConditions] a WITH (NOLOCK)

WHERE
        ISNULL(TRY_CAST(a.[Пк7] AS int),0) = 1
    AND CAST(a.DT_OPEN_FACT AS date)
            BETWEEN @OpenFrom AND @OpenTo
    AND a.CLI_ID IS NOT NULL;


CREATE INDEX IX_pk7_openings_cli
ON #pk7_openings(cli_id);

CREATE INDEX IX_pk7_openings_con
ON #pk7_openings(con_id);



SELECT DISTINCT
    cli_id

INTO #pk7_clients

FROM #pk7_openings

WHERE cli_id IS NOT NULL;


CREATE UNIQUE CLUSTERED INDEX IX_pk7_clients
ON #pk7_clients(cli_id);



/* ============================================================
   РЕЗУЛЬТАТ 0
   КАКИЕ ИМЕННО ВКЛАДЫ С Пк7 СФОРМИРОВАЛИ КОГОРТУ
   ============================================================ */

SELECT
      cli_id
    , con_id
    , dt_open_fact
    , dt_close_plan
    , dt_close_fact
    , START_DEPOSIT
    , DEPOSIT_ADD_CONDITIONS
    , PROMO_CODE
    , PROMO_GROUP

FROM #pk7_openings

ORDER BY
      cli_id
    , dt_open_fact
    , con_id;



/* ============================================================
   2. ИСТОРИЯ БАЛАНСА ЭТИХ КЛИЕНТОВ
      01.07.2026 - 22.09.2026

      Один клиент / одна дата:
      - срочные
      - НС
      - общий баланс
   ============================================================ */

SELECT
      b.dt_rep
    , CAST(b.cli_id AS bigint) AS cli_id

    , SUM(
        CASE
            WHEN b.section_name = N'Срочные'
                THEN b.out_rub
            ELSE 0
        END
      ) AS td_sum

    , SUM(
        CASE
            WHEN b.section_name = N'Накопительный счёт'
                THEN b.out_rub
            ELSE 0
        END
      ) AS ns_sum

    , SUM(b.out_rub) AS total_sum

FROM [ALM].[ALM].[VW_balance_rest_all] b WITH (NOLOCK)

INNER JOIN #pk7_clients c
    ON c.cli_id = CAST(b.cli_id AS bigint)

WHERE
        b.dt_rep BETWEEN @HistoryFrom AND @BaseDate

    AND b.section_name IN
        (
            N'Срочные',
            N'Накопительный счёт'
        )

    AND b.block_name = N'Привлечение ФЛ'
    AND b.acc_role   = N'LIAB'
    AND b.od_flag    = 1
    AND b.cur        = '810'

    AND b.out_rub IS NOT NULL
    AND b.out_rub >= 0

GROUP BY
      b.dt_rep
    , CAST(b.cli_id AS bigint)

ORDER BY
      cli_id
    , dt_rep

OPTION (RECOMPILE);



/* ============================================================
   3. SNAPSHOT НА 22.09
   ============================================================ */

SELECT
      CAST(b.cli_id AS bigint)          AS cli_id
    , CAST(b.con_id AS bigint)          AS con_id

    , CAST(b.dt_open AS date)           AS dt_open
    , CAST(b.dt_close_plan AS date)     AS dt_close_plan

    , b.section_name
    , b.PROD_NAME_res
    , b.TSEGMENTNAME

    , CAST(b.out_rub AS decimal(38,2))  AS out_rub

    , b.rate_con
    , b.termdays

INTO #bal_2209

FROM [ALM].[ALM].[VW_balance_rest_all] b WITH (NOLOCK)

INNER JOIN #pk7_clients c
    ON c.cli_id = CAST(b.cli_id AS bigint)

WHERE
        b.dt_rep = @BaseDate

    AND b.section_name IN
        (
            N'Срочные',
            N'Накопительный счёт'
        )

    AND b.block_name = N'Привлечение ФЛ'
    AND b.acc_role   = N'LIAB'
    AND b.od_flag    = 1
    AND b.cur        = '810'

    AND b.out_rub IS NOT NULL
    AND b.out_rub >= 0

OPTION (RECOMPILE);


CREATE INDEX IX_bal2209_cli
ON #bal_2209(cli_id);

CREATE INDEX IX_bal2209_con
ON #bal_2209(con_id);



/* ============================================================
   4. SNAPSHOT НА 01.10
   ============================================================ */

SELECT
      CAST(b.cli_id AS bigint)          AS cli_id
    , CAST(b.con_id AS bigint)          AS con_id

    , CAST(b.dt_open AS date)           AS dt_open
    , CAST(b.dt_close_plan AS date)     AS dt_close_plan

    , b.section_name
    , b.PROD_NAME_res
    , b.TSEGMENTNAME

    , CAST(b.out_rub AS decimal(38,2))  AS out_rub

    , b.rate_con
    , b.termdays

INTO #bal_0110

FROM [ALM].[ALM].[VW_balance_rest_all] b WITH (NOLOCK)

INNER JOIN #pk7_clients c
    ON c.cli_id = CAST(b.cli_id AS bigint)

WHERE
        b.dt_rep = @AnalysisDate

    AND b.section_name IN
        (
            N'Срочные',
            N'Накопительный счёт'
        )

    AND b.block_name = N'Привлечение ФЛ'
    AND b.acc_role   = N'LIAB'
    AND b.od_flag    = 1
    AND b.cur        = '810'

    AND b.out_rub IS NOT NULL
    AND b.out_rub >= 0

OPTION (RECOMPILE);


CREATE INDEX IX_bal0110_cli
ON #bal_0110(cli_id);

CREATE INDEX IX_bal0110_con
ON #bal_0110(con_id);



/* ============================================================
   5. ВСЕ НУЖНЫЕ CON_ID
   ============================================================ */

SELECT con_id

INTO #relevant_con_ids

FROM
(
    SELECT con_id
    FROM #bal_2209
    WHERE con_id IS NOT NULL

    UNION

    SELECT con_id
    FROM #bal_0110
    WHERE con_id IS NOT NULL

    UNION

    SELECT con_id
    FROM #pk7_openings
    WHERE con_id IS NOT NULL
) q;


CREATE UNIQUE CLUSTERED INDEX IX_relevant_con_ids
ON #relevant_con_ids(con_id);



/* ============================================================
   6. ПОСЛЕДНЯЯ ЗАПИСЬ ПО НАДБАВКАМ / УСЛОВИЯМ ВКЛАДА
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
   7. ВКЛАДЫ К ВЫХОДУ НА 22.09

      Вклад существует на 22.09
      и планово заканчивается 22.09-01.10
   ============================================================ */

SELECT
      b.cli_id
    , b.con_id

    , b.dt_open
    , b.dt_close_plan

    , DATEDIFF(day,b.dt_open,b.dt_close_plan) AS term_days_calc
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
    AND b.dt_close_plan BETWEEN @BaseDate AND @AnalysisDate;



/* ============================================================
   РЕЗУЛЬТАТ 1
   ПОКОНТРАКТНАЯ ВЫГРУЗКА ВКЛАДОВ К ВЫХОДУ
   ============================================================ */

SELECT *
FROM #exit_contracts
ORDER BY
      cli_id
    , dt_close_plan
    , con_id;



/* ============================================================
   РЕЗУЛЬТАТ 2
   ПОКЛИЕНТНЫЙ СРЕЗ НА 22.09

   - был ли вклад к выходу
   - объём вкладов к выходу
   - количество вкладов к выходу
   - есть ли ненулевой НС
   - остаток НС
   - прочие вклады
   ============================================================ */

;WITH exit_agg AS
(
    SELECT
          cli_id
        , COUNT(DISTINCT con_id) AS exit_td_count
        , SUM(out_rub) AS exit_td_sum

    FROM #exit_contracts

    GROUP BY cli_id
),

ns AS
(
    SELECT
          cli_id
        , SUM(out_rub) AS ns_sum

    FROM #bal_2209

    WHERE section_name = N'Накопительный счёт'

    GROUP BY cli_id
),

other_td AS
(
    SELECT
          cli_id

        , SUM(
            CASE
                WHEN NOT (
                        dt_close_plan BETWEEN @BaseDate
                                          AND @AnalysisDate
                    )
                THEN out_rub
                ELSE 0
            END
          ) AS other_td_sum

    FROM #bal_2209

    WHERE section_name = N'Срочные'

    GROUP BY cli_id
)

SELECT
      c.cli_id

    , CASE
        WHEN e.cli_id IS NOT NULL THEN 1
        ELSE 0
      END AS has_exit_td_flag

    , ISNULL(e.exit_td_count,0) AS exit_td_count
    , ISNULL(e.exit_td_sum,0)   AS exit_td_sum

    , CASE
        WHEN ISNULL(n.ns_sum,0) > 0 THEN 1
        ELSE 0
      END AS has_ns_nonzero_flag

    , ISNULL(n.ns_sum,0) AS ns_sum

    , ISNULL(o.other_td_sum,0) AS other_td_sum

FROM #pk7_clients c

LEFT JOIN exit_agg e
    ON e.cli_id = c.cli_id

LEFT JOIN ns n
    ON n.cli_id = c.cli_id

LEFT JOIN other_td o
    ON o.cli_id = c.cli_id

ORDER BY c.cli_id;



/* ============================================================
   РЕЗУЛЬТАТ 3
   СОСТОЯНИЕ НА 01.10

   Для каждого клиента:
   - сколько новых вкладов Пк7
   - объём новых вкладов Пк7
   - сколько остальных новых вкладов
   - объём остальных новых вкладов
   - всего новых вкладов
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
        AND b.dt_open BETWEEN @OpenFrom AND @OpenTo

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

    WHERE section_name = N'Накопительный счёт'

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

ORDER BY c.cli_id;



/* ============================================================
   CLEANUP
   ============================================================ */

DROP TABLE IF EXISTS #exit_contracts;
DROP TABLE IF EXISTS #attr_latest;
DROP TABLE IF EXISTS #relevant_con_ids;
DROP TABLE IF EXISTS #bal_0110;
DROP TABLE IF EXISTS #bal_2209;
DROP TABLE IF EXISTS #pk7_openings;
DROP TABLE IF EXISTS #pk7_clients;
