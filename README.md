USE [ALM];
SET NOCOUNT ON;


/* ============================================================
   0. ПАРАМЕТРЫ

   Анализ:
       июнь  : 31.05 -> 30.06
       июль  : 30.06 -> 31.07
       август: 31.07 -> 31.08
   ============================================================ */

DECLARE @StartBaseDate date = '2026-05-31';
DECLARE @LastObservationDate date = '2026-08-31';


IF @LastObservationDate <= @StartBaseDate
    THROW 51000,
          N'Последняя дата должна быть позже начальной даты баланса.',
          1;


/* ============================================================
   Очистка
   ============================================================ */

IF OBJECT_ID('tempdb..#periods') IS NOT NULL DROP TABLE #periods;
IF OBJECT_ID('tempdb..#snapshot_dates') IS NOT NULL DROP TABLE #snapshot_dates;
IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;
IF OBJECT_ID('tempdb..#attr_flags') IS NOT NULL DROP TABLE #attr_flags;
IF OBJECT_ID('tempdb..#exit_classified') IS NOT NULL DROP TABLE #exit_classified;
IF OBJECT_ID('tempdb..#opened_classified') IS NOT NULL DROP TABLE #opened_classified;
IF OBJECT_ID('tempdb..#cohort_clients') IS NOT NULL DROP TABLE #cohort_clients;
IF OBJECT_ID('tempdb..#client_attr_flags') IS NOT NULL DROP TABLE #client_attr_flags;
IF OBJECT_ID('tempdb..#client_panel') IS NOT NULL DROP TABLE #client_panel;
IF OBJECT_ID('tempdb..#client_mart') IS NOT NULL DROP TABLE #client_mart;


/* ============================================================
   1. АВТОМАТИЧЕСКИ СОБИРАЕМ ПЕРИОДЫ НАБЛЮДЕНИЯ
   ============================================================ */

CREATE TABLE #periods
(
      period_id         int  NOT NULL
    , observation_month date NOT NULL
    , base_date         date NOT NULL
    , end_date          date NOT NULL
    , exit_from         date NOT NULL
    , exit_to           date NOT NULL
    , open_from         date NOT NULL
    , open_to           date NOT NULL
);


DECLARE @LoopBaseDate date = @StartBaseDate;
DECLARE @LoopEndDate date;
DECLARE @PeriodId int = 1;


WHILE @LoopBaseDate < @LastObservationDate
BEGIN

    SET @LoopEndDate =
        EOMONTH(
            DATEADD(month, 1, @LoopBaseDate)
        );

    IF @LoopEndDate > @LastObservationDate
        SET @LoopEndDate = @LastObservationDate;


    INSERT INTO #periods
    (
          period_id
        , observation_month
        , base_date
        , end_date
        , exit_from
        , exit_to
        , open_from
        , open_to
    )
    VALUES
    (
          @PeriodId
        , DATEFROMPARTS(
              YEAR(@LoopEndDate),
              MONTH(@LoopEndDate),
              1
          )
        , @LoopBaseDate
        , @LoopEndDate
        , DATEADD(day, 1, @LoopBaseDate)
        , @LoopEndDate
        , DATEADD(day, 1, @LoopBaseDate)
        , @LoopEndDate
    );


    SET @LoopBaseDate = @LoopEndDate;
    SET @PeriodId = @PeriodId + 1;

END;


/* ============================================================
   2. НУЖНЫЕ SNAPSHOT-ДАТЫ
   ============================================================ */

SELECT
    x.dt_rep
INTO #snapshot_dates
FROM
(
    SELECT base_date AS dt_rep
    FROM #periods

    UNION

    SELECT end_date AS dt_rep
    FROM #periods
) x;


/* ============================================================
   3. БАЛАНС ТОЛЬКО НА НУЖНЫЕ ДАТЫ

   Каждая дата вызывается отдельно:
       WHERE dt_rep = @SnapshotDate
   ============================================================ */

CREATE TABLE #bal
(
      dt_rep date NOT NULL
    , cli_id bigint NOT NULL
    , con_id bigint NULL
    , dt_open date NULL
    , dt_close_plan date NULL
    , section_name nvarchar(500) NULL
    , PROD_NAME_res nvarchar(500) NULL
    , out_rub decimal(38,6) NOT NULL
    , TSEGMENTNAME nvarchar(500) NULL
);


DECLARE @SnapshotDate date;


SELECT
    @SnapshotDate = MIN(dt_rep)
FROM #snapshot_dates;


WHILE @SnapshotDate IS NOT NULL
BEGIN

    INSERT INTO #bal
    (
          dt_rep
        , cli_id
        , con_id
        , dt_open
        , dt_close_plan
        , section_name
        , PROD_NAME_res
        , out_rub
        , TSEGMENTNAME
    )

    SELECT
          CAST(t.dt_rep AS date)
        , CAST(t.cli_id AS bigint)
        , CAST(t.con_id AS bigint)
        , CAST(t.dt_open AS date)
        , CAST(t.dt_close_plan AS date)
        , t.section_name
        , t.PROD_NAME_res
        , CAST(t.out_rub AS decimal(38,6))
        , t.TSEGMENTNAME

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    WHERE
        t.dt_rep = @SnapshotDate

        AND t.section_name IN
        (
              N'Срочные'
            , N'Накопительный счёт'
        )

        AND t.block_name = N'Привлечение ФЛ'
        AND t.acc_role   = N'LIAB'
        AND t.od_flag    = 1
        AND t.cur        = '810'

        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0

    OPTION (RECOMPILE);


    SELECT
        @SnapshotDate = MIN(dt_rep)
    FROM #snapshot_dates
    WHERE dt_rep > @SnapshotDate;

END;


/* ============================================================
   4. ПРИЗНАКИ ДОГОВОРОВ
   ============================================================ */

WITH relevant_con_id AS
(
    SELECT DISTINCT
        con_id
    FROM #bal
    WHERE con_id IS NOT NULL
),

attr_ranked AS
(
    SELECT
          CAST(a.CON_ID AS bigint) AS con_id

        /* Нов */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Нов] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_nov_flag

        /* Пр2 / Пр3 */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Пр2] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Пр3] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_pr2_pr3_flag

        /* НДП / НДМ */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[НДП] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[НДМ] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_ndp_ndm_flag

        /* Пк2 / От1 */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Пк2] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[От1] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_pk2_ot1_flag

        /* Пк3 / Пк6 */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Пк3] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Пк6] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_pk3_pk6_flag

        /* МПЛ */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Мпл] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_mpl_flag

        /* Пнс */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Пнс] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_pns_flag

        , ROW_NUMBER() OVER
          (
              PARTITION BY a.CON_ID
              ORDER BY
                    a.DT_UPDATE DESC
                  , a.loaddate DESC
          ) AS rn

    FROM alm.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    INNER JOIN relevant_con_id r
        ON r.con_id = CAST(a.CON_ID AS bigint)
)

SELECT
      con_id
    , is_nov_flag
    , is_pr2_pr3_flag
    , is_ndp_ndm_flag
    , is_pk2_ot1_flag
    , is_pk3_pk6_flag
    , is_mpl_flag
    , is_pns_flag
INTO #attr_flags
FROM attr_ranked
WHERE rn = 1;


/* ============================================================
   5. ВСЕ ВКЛАДЫ К ВЫХОДУ ПО ПЕРИОДАМ
   ============================================================ */

WITH exit_by_con AS
(
    SELECT
          p.period_id
        , p.observation_month

        , b.cli_id
        , b.con_id

        , MIN(b.dt_open) AS dt_open
        , SUM(b.out_rub) AS out_rub


        /* ФУ */
        , MAX
          (
              CASE
                  WHEN b.PROD_NAME_res IN
                  (
                        N'Надёжный прайм'
                      , N'Надёжный VIP'
                      , N'Надёжный премиум'
                      , N'Надёжный промо'
                      , N'Надёжный старт'
                      , N'Надёжный Т2'
                      , N'Надёжный Мегафон'
                      , N'Надёжный процент'
                      , N'Надёжныйпроцент'
                      , N'Могучий'
                      , N'Надёжный'
                  )
                      THEN 1
                  ELSE 0
              END
          ) AS is_fu_flag

        , MAX(ISNULL(a.is_nov_flag, 0))
            AS is_nov_flag

        , MAX(ISNULL(a.is_pr2_pr3_flag, 0))
            AS is_pr2_pr3_flag

        , MAX(ISNULL(a.is_ndp_ndm_flag, 0))
            AS is_ndp_ndm_flag

        , MAX(ISNULL(a.is_pk2_ot1_flag, 0))
            AS is_pk2_ot1_flag

        , MAX(ISNULL(a.is_pk3_pk6_flag, 0))
            AS is_pk3_pk6_flag

        , MAX(ISNULL(a.is_mpl_flag, 0))
            AS is_mpl_flag

        , MAX(ISNULL(a.is_pns_flag, 0))
            AS is_pns_flag

    FROM #periods p

    INNER JOIN #bal b
        ON b.dt_rep = p.base_date

    LEFT JOIN #attr_flags a
        ON a.con_id = b.con_id

    WHERE
        b.section_name = N'Срочные'

        AND b.dt_close_plan >= p.exit_from
        AND b.dt_close_plan <= p.exit_to

    GROUP BY
          p.period_id
        , p.observation_month
        , b.cli_id
        , b.con_id
)

SELECT
      e.period_id
    , e.observation_month
    , e.cli_id
    , e.con_id
    , e.out_rub

    , CASE
          WHEN e.is_fu_flag = 1
              THEN N'fu'

          WHEN e.is_nov_flag = 1
              THEN N'nov'

          WHEN e.is_pr2_pr3_flag = 1
              THEN N'pr2_pr3'

          WHEN e.is_ndp_ndm_flag = 1
              THEN N'ndp_ndm'

          WHEN e.is_pk2_ot1_flag = 1
              THEN N'pk2_ot1'

          WHEN e.is_pk3_pk6_flag = 1
              THEN N'pk3_pk6'

          WHEN e.is_mpl_flag = 1
              THEN N'mpl'

          WHEN e.is_pns_flag = 1
              THEN N'pns'

          ELSE N'other'

      END AS exit_category

INTO #exit_classified
FROM exit_by_con e;


/* ============================================================
   6. КОГОРТА

   Только клиенты, у которых был Пк2 / От1 к выходу.

   cohort_month =
   ПЕРВЫЙ такой месяц.
   ============================================================ */

SELECT
      e.cli_id
    , MIN(e.observation_month) AS cohort_month
INTO #cohort_clients
FROM #exit_classified e
WHERE
    e.exit_category = N'pk2_ot1'
GROUP BY
    e.cli_id;


/* ============================================================
   7. НОВЫЕ КЛИЕНТСКИЕ ФЛАГИ

   Нас интересует сам факт, что в промежутке
   @StartBaseDate ... @LastObservationDate

   существовал хотя бы один период соответствующего атрибута.

   Правило пересечения периодов:

       dt_from <= конец анализа
       AND
       dt_to >= начало анализа

   NULL dt_to также считаем действующим периодом.

   4444-01-01 в dt_to автоматически считается
   периодом, продолжающимся далеко в будущее.
   ============================================================ */

SELECT
      c.cli_id

    , CAST(
          MAX(
              CASE
                  WHEN s.cli_attr_code = N'PREMIUM_PACKAGE'
                      THEN 1
                  ELSE 0
              END
          )
          AS bit
      ) AS has_premium_package_flag

    , CAST(
          MAX(
              CASE
                  WHEN s.cli_attr_code = N'IS_SALARY_CLIENT'
                      THEN 1
                  ELSE 0
              END
          )
          AS bit
      ) AS has_salary_client_flag

INTO #client_attr_flags

FROM #cohort_clients c

LEFT JOIN alm.ehd.attr_cli_stat_sal s WITH (NOLOCK)
    ON s.cli_id = c.cli_id

   AND s.cli_attr_code IN
   (
         N'PREMIUM_PACKAGE'
       , N'IS_SALARY_CLIENT'
   )

   /* Запись должна пересекаться с нашим периодом анализа */
   AND s.dt_from < DATEADD(day, 1, @LastObservationDate)

   AND
   (
       s.dt_to IS NULL
       OR s.dt_to >= @StartBaseDate
   )

GROUP BY
    c.cli_id;


/* ============================================================
   8. ПАНЕЛЬ КЛИЕНТОВ

   Клиент находится во всех observation_month,
   начиная со своей первой когорты.
   ============================================================ */

SELECT
      c.cli_id
    , c.cohort_month

    , p.period_id
    , p.observation_month

    , p.base_date
    , p.end_date

    , p.exit_from
    , p.exit_to

    , p.open_from
    , p.open_to

INTO #client_panel

FROM #cohort_clients c

CROSS JOIN #periods p

WHERE
    p.observation_month >= c.cohort_month;


/* ============================================================
   9. ОТКРЫТЫЕ ВКЛАДЫ КОГОРТНЫХ КЛИЕНТОВ
   ============================================================ */

WITH opened_by_con AS
(
    SELECT
          p.period_id
        , p.observation_month

        , b.cli_id
        , b.con_id

        , SUM(b.out_rub) AS out_rub


        /* ФУ */
        , MAX
          (
              CASE
                  WHEN b.PROD_NAME_res IN
                  (
                        N'Надёжный прайм'
                      , N'Надёжный VIP'
                      , N'Надёжный премиум'
                      , N'Надёжный промо'
                      , N'Надёжный старт'
                      , N'Надёжный Т2'
                      , N'Надёжный Мегафон'
                      , N'Надёжный процент'
                      , N'Надёжныйпроцент'
                      , N'Могучий'
                      , N'Надёжный'
                  )
                      THEN 1
                  ELSE 0
              END
          ) AS is_fu_flag

        , MAX(ISNULL(a.is_nov_flag, 0))
            AS is_nov_flag

        , MAX(ISNULL(a.is_pr2_pr3_flag, 0))
            AS is_pr2_pr3_flag

        , MAX(ISNULL(a.is_ndp_ndm_flag, 0))
            AS is_ndp_ndm_flag

        , MAX(ISNULL(a.is_pk2_ot1_flag, 0))
            AS is_pk2_ot1_flag

        , MAX(ISNULL(a.is_pk3_pk6_flag, 0))
            AS is_pk3_pk6_flag

        , MAX(ISNULL(a.is_mpl_flag, 0))
            AS is_mpl_flag

        , MAX(ISNULL(a.is_pns_flag, 0))
            AS is_pns_flag

    FROM #periods p

    INNER JOIN #bal b
        ON b.dt_rep = p.end_date

    INNER JOIN #cohort_clients c
        ON c.cli_id = b.cli_id

    LEFT JOIN #attr_flags a
        ON a.con_id = b.con_id

    WHERE
        b.section_name = N'Срочные'

        AND b.dt_open >= p.open_from
        AND b.dt_open <= p.open_to

        AND p.observation_month >= c.cohort_month

    GROUP BY
          p.period_id
        , p.observation_month
        , b.cli_id
        , b.con_id
)

SELECT
      o.period_id
    , o.observation_month

    , o.cli_id
    , o.con_id
    , o.out_rub

    , CASE
          WHEN o.is_fu_flag = 1
              THEN N'fu'

          WHEN o.is_nov_flag = 1
              THEN N'nov'

          WHEN o.is_pr2_pr3_flag = 1
              THEN N'pr2_pr3'

          WHEN o.is_ndp_ndm_flag = 1
              THEN N'ndp_ndm'

          WHEN o.is_pk2_ot1_flag = 1
              THEN N'pk2_ot1'

          WHEN o.is_pk3_pk6_flag = 1
              THEN N'pk3_pk6'

          WHEN o.is_mpl_flag = 1
              THEN N'mpl'

          WHEN o.is_pns_flag = 1
              THEN N'pns'

          ELSE N'other'

      END AS open_category

INTO #opened_classified

FROM opened_by_con o;


/* ============================================================
   10. СОБИРАЕМ ПОКЛИЕНТНУЮ ВИТРИНУ
   ============================================================ */

WITH base_state AS
(
    SELECT
          p.period_id
        , b.cli_id

        , MAX
          (
              CASE
                  WHEN b.TSEGMENTNAME = N'ДЧБО'
                      THEN 1
                  ELSE 0
              END
          ) AS is_dchbo_client

        , MAX
          (
              CASE
                  WHEN b.section_name = N'Накопительный счёт'
                      THEN 1
                  ELSE 0
              END
          ) AS has_ns_base_flag

        , SUM
          (
              CASE
                  WHEN b.section_name = N'Накопительный счёт'
                      THEN b.out_rub
                  ELSE 0
              END
          ) AS ns_start_sum

    FROM #periods p

    INNER JOIN #bal b
        ON b.dt_rep = p.base_date

    INNER JOIN #cohort_clients c
        ON c.cli_id = b.cli_id

    GROUP BY
          p.period_id
        , b.cli_id
),


end_state AS
(
    SELECT
          p.period_id
        , b.cli_id

        , SUM
          (
              CASE
                  WHEN b.section_name = N'Накопительный счёт'
                      THEN b.out_rub
                  ELSE 0
              END
          ) AS ns_end_sum

    FROM #periods p

    INNER JOIN #bal b
        ON b.dt_rep = p.end_date

    INNER JOIN #cohort_clients c
        ON c.cli_id = b.cli_id

    GROUP BY
          p.period_id
        , b.cli_id
),


other_td_flag AS
(
    SELECT
          cp.period_id
        , cp.cli_id

        , MAX
          (
              CASE
                  WHEN b.section_name = N'Срочные'

                       AND NOT
                       (
                               b.dt_close_plan >= cp.exit_from
                           AND b.dt_close_plan <= cp.exit_to
                       )

                      THEN 1

                  ELSE 0
              END
          ) AS has_other_rub_td_flag

    FROM #client_panel cp

    LEFT JOIN #bal b
        ON b.cli_id = cp.cli_id
       AND b.dt_rep = cp.base_date

    GROUP BY
          cp.period_id
        , cp.cli_id
),


sms_clients AS
(
    SELECT DISTINCT
          p.period_id
        , CAST(m.cli_id AS bigint) AS cli_id

    FROM #periods p

    INNER JOIN alm.ehd.ODS_058_VI_MESSAGE2DWH m WITH (NOLOCK)
        ON m.msgbegindate >= p.open_from
       AND m.msgbegindate < DATEADD(day, 1, p.open_to)

    INNER JOIN #cohort_clients c
        ON c.cli_id = CAST(m.cli_id AS bigint)

    WHERE
        m.cli_id IS NOT NULL
),


/* ============================================================
   ВКЛАДЫ К ВЫХОДУ:
   объёмы + клиентские флаги
   ============================================================ */

exit_agg AS
(
    SELECT
          e.period_id
        , e.cli_id

        , SUM(e.out_rub)
            AS exit_td_sum

        , CAST(1 AS int)
            AS has_any_exit_td_flag


        /* ФУ */
        , SUM(
              CASE WHEN e.exit_category = N'fu'
                   THEN e.out_rub ELSE 0 END
          ) AS exit_fu_td_sum

        , MAX(
              CASE WHEN e.exit_category = N'fu'
                   THEN 1 ELSE 0 END
          ) AS has_fu_exit_td_flag


        /* НОВ */
        , SUM(
              CASE WHEN e.exit_category = N'nov'
                   THEN e.out_rub ELSE 0 END
          ) AS exit_nov_td_sum

        , MAX(
              CASE WHEN e.exit_category = N'nov'
                   THEN 1 ELSE 0 END
          ) AS has_nov_exit_td_flag


        /* Пр2 / Пр3 */
        , SUM(
              CASE WHEN e.exit_category = N'pr2_pr3'
                   THEN e.out_rub ELSE 0 END
          ) AS exit_pr2_pr3_td_sum

        , MAX(
              CASE WHEN e.exit_category = N'pr2_pr3'
                   THEN 1 ELSE 0 END
          ) AS has_pr2_pr3_exit_td_flag


        /* НДП / НДМ */
        , SUM(
              CASE WHEN e.exit_category = N'ndp_ndm'
                   THEN e.out_rub ELSE 0 END
          ) AS exit_ndp_ndm_td_sum

        , MAX(
              CASE WHEN e.exit_category = N'ndp_ndm'
                   THEN 1 ELSE 0 END
          ) AS has_ndp_ndm_exit_td_flag


        /* Пк2 / От1 */
        , SUM(
              CASE WHEN e.exit_category = N'pk2_ot1'
                   THEN e.out_rub ELSE 0 END
          ) AS exit_pk2_ot1_td_sum

        , MAX(
              CASE WHEN e.exit_category = N'pk2_ot1'
                   THEN 1 ELSE 0 END
          ) AS has_pk2_ot1_exit_td_flag


        /* Пк3 / Пк6 */
        , SUM(
              CASE WHEN e.exit_category = N'pk3_pk6'
                   THEN e.out_rub ELSE 0 END
          ) AS exit_pk3_pk6_td_sum

        , MAX(
              CASE WHEN e.exit_category = N'pk3_pk6'
                   THEN 1 ELSE 0 END
          ) AS has_pk3_pk6_exit_td_flag


        /* МПЛ */
        , SUM(
              CASE WHEN e.exit_category = N'mpl'
                   THEN e.out_rub ELSE 0 END
          ) AS exit_mpl_td_sum

        , MAX(
              CASE WHEN e.exit_category = N'mpl'
                   THEN 1 ELSE 0 END
          ) AS has_mpl_exit_td_flag


        /* Пнс */
        , SUM(
              CASE WHEN e.exit_category = N'pns'
                   THEN e.out_rub ELSE 0 END
          ) AS exit_pns_td_sum

        , MAX(
              CASE WHEN e.exit_category = N'pns'
                   THEN 1 ELSE 0 END
          ) AS has_pns_exit_td_flag


        /* Остальные */
        , SUM(
              CASE WHEN e.exit_category = N'other'
                   THEN e.out_rub ELSE 0 END
          ) AS exit_other_td_sum


    FROM #exit_classified e

    INNER JOIN #cohort_clients c
        ON c.cli_id = e.cli_id
       AND e.observation_month >= c.cohort_month

    GROUP BY
          e.period_id
        , e.cli_id
),


/* ============================================================
   ОТКРЫТЫЕ ВКЛАДЫ:
   объёмы + клиентские флаги
   ============================================================ */

opened_agg AS
(
    SELECT
          o.period_id
        , o.cli_id


        /* ФУ */
        , SUM(
              CASE WHEN o.open_category = N'fu'
                   THEN o.out_rub ELSE 0 END
          ) AS opened_fu

        , MAX(
              CASE WHEN o.open_category = N'fu'
                   THEN 1 ELSE 0 END
          ) AS has_opened_fu_flag


        /* НОВ */
        , SUM(
              CASE WHEN o.open_category = N'nov'
                   THEN o.out_rub ELSE 0 END
          ) AS opened_nov

        , MAX(
              CASE WHEN o.open_category = N'nov'
                   THEN 1 ELSE 0 END
          ) AS has_opened_nov_flag


        /* Пр2 / Пр3 */
        , SUM(
              CASE WHEN o.open_category = N'pr2_pr3'
                   THEN o.out_rub ELSE 0 END
          ) AS opened_pr2_pr3

        , MAX(
              CASE WHEN o.open_category = N'pr2_pr3'
                   THEN 1 ELSE 0 END
          ) AS has_opened_pr2_pr3_flag


        /* НДП / НДМ */
        , SUM(
              CASE WHEN o.open_category = N'ndp_ndm'
                   THEN o.out_rub ELSE 0 END
          ) AS opened_ndp_ndm

        , MAX(
              CASE WHEN o.open_category = N'ndp_ndm'
                   THEN 1 ELSE 0 END
          ) AS has_opened_ndp_ndm_flag


        /* Пк2 / От1 */
        , SUM(
              CASE WHEN o.open_category = N'pk2_ot1'
                   THEN o.out_rub ELSE 0 END
          ) AS opened_pk2_ot1

        , MAX(
              CASE WHEN o.open_category = N'pk2_ot1'
                   THEN 1 ELSE 0 END
          ) AS has_opened_pk2_ot1_flag


        /* Пк3 / Пк6 */
        , SUM(
              CASE WHEN o.open_category = N'pk3_pk6'
                   THEN o.out_rub ELSE 0 END
          ) AS opened_pk3_pk6

        , MAX(
              CASE WHEN o.open_category = N'pk3_pk6'
                   THEN 1 ELSE 0 END
          ) AS has_opened_pk3_pk6_flag


        /* МПЛ */
        , SUM(
              CASE WHEN o.open_category = N'mpl'
                   THEN o.out_rub ELSE 0 END
          ) AS opened_mpl

        , MAX(
              CASE WHEN o.open_category = N'mpl'
                   THEN 1 ELSE 0 END
          ) AS has_opened_mpl_flag


        /* Пнс */
        , SUM(
              CASE WHEN o.open_category = N'pns'
                   THEN o.out_rub ELSE 0 END
          ) AS opened_pns

        , MAX(
              CASE WHEN o.open_category = N'pns'
                   THEN 1 ELSE 0 END
          ) AS has_opened_pns_flag


        /* Остальные */
        , SUM(
              CASE WHEN o.open_category = N'other'
                   THEN o.out_rub ELSE 0 END
          ) AS opened_other


        /* Все открытия */
        , SUM(o.out_rub)
            AS opened_total


    FROM #opened_classified o

    GROUP BY
          o.period_id
        , o.cli_id
)


/* ============================================================
   11. ФИНАЛЬНАЯ ПОКЛИЕНТНАЯ ПАНЕЛЬ
   ============================================================ */

SELECT
      cp.cli_id

    , cp.cohort_month
    , cp.observation_month


    /* ========================================================
       НОВЫЕ ФЛАГИ КЛИЕНТА
       ======================================================== */

    , ISNULL(caf.has_premium_package_flag, 0)
        AS has_premium_package_flag

    , ISNULL(caf.has_salary_client_flag, 0)
        AS has_salary_client_flag


    /* Тип клиента в текущем observation_month */

    , CASE
          WHEN ISNULL(e.has_any_exit_td_flag, 0) = 1
              THEN N'01. Вкладчики к выходу'

          WHEN ISNULL(bs.has_ns_base_flag, 0) = 1
              THEN N'02. НС без вкладов к выходу'

          ELSE N'03. Нет вкладов к выходу и НС'

      END AS client_base_type


    /* Сегмент */
    , CASE
          WHEN ISNULL(bs.is_dchbo_client, 0) = 1
              THEN N'ДЧБО'

          ELSE N'Розница'

      END AS segment_flag


    /* SMS */
    , CASE
          WHEN sms.cli_id IS NOT NULL
              THEN 1
          ELSE 0
      END AS is_sms_sent


    /* ========================================================
       Бакет
       ======================================================== */

    , CASE

          WHEN ISNULL(e.has_any_exit_td_flag, 0) = 1
               AND ISNULL(e.exit_td_sum, 0) <= 1000000
              THEN N'1. Выход|НС <= 1.0 млн'

          WHEN ISNULL(e.has_any_exit_td_flag, 0) = 1
               AND ISNULL(e.exit_td_sum, 0) < 5000000
              THEN N'2. Выход|НС 1.0-5 млн'

          WHEN ISNULL(e.has_any_exit_td_flag, 0) = 1
              THEN N'3. Выход|НС >= 5 млн'


          WHEN ISNULL(e.has_any_exit_td_flag, 0) = 0
               AND ISNULL(bs.has_ns_base_flag, 0) = 1
               AND ISNULL(bs.ns_start_sum, 0) <= 1000000
              THEN N'1. Выход|НС <= 1.0 млн'

          WHEN ISNULL(e.has_any_exit_td_flag, 0) = 0
               AND ISNULL(bs.has_ns_base_flag, 0) = 1
               AND ISNULL(bs.ns_start_sum, 0) < 5000000
              THEN N'2. Выход|НС 1.0-5 млн'

          WHEN ISNULL(e.has_any_exit_td_flag, 0) = 0
               AND ISNULL(bs.has_ns_base_flag, 0) = 1
              THEN N'3. Выход|НС >= 5 млн'

          ELSE N'Не определено'

      END AS base_amount_flag


    /* ========================================================
       ВКЛАДЫ К ВЫХОДУ — ОБЪЁМЫ
       ======================================================== */

    , ISNULL(e.exit_td_sum, 0)
        AS exit_td_sum

    , ISNULL(e.exit_fu_td_sum, 0)
        AS exit_fu_td_sum

    , ISNULL(e.exit_nov_td_sum, 0)
        AS exit_nov_td_sum

    , ISNULL(e.exit_pr2_pr3_td_sum, 0)
        AS exit_pr2_pr3_td_sum

    , ISNULL(e.exit_ndp_ndm_td_sum, 0)
        AS exit_ndp_ndm_td_sum

    , ISNULL(e.exit_pk2_ot1_td_sum, 0)
        AS exit_pk2_ot1_td_sum

    , ISNULL(e.exit_pk3_pk6_td_sum, 0)
        AS exit_pk3_pk6_td_sum

    , ISNULL(e.exit_mpl_td_sum, 0)
        AS exit_mpl_td_sum

    , ISNULL(e.exit_pns_td_sum, 0)
        AS exit_pns_td_sum

    , ISNULL(e.exit_other_td_sum, 0)
        AS exit_other_td_sum


    /* ========================================================
       ВКЛАДЫ К ВЫХОДУ — ФЛАГИ
       ======================================================== */

    , ISNULL(e.has_fu_exit_td_flag, 0)
        AS has_fu_exit_td_flag

    , ISNULL(e.has_nov_exit_td_flag, 0)
        AS has_nov_exit_td_flag

    , ISNULL(e.has_pr2_pr3_exit_td_flag, 0)
        AS has_pr2_pr3_exit_td_flag

    , ISNULL(e.has_ndp_ndm_exit_td_flag, 0)
        AS has_ndp_ndm_exit_td_flag

    , ISNULL(e.has_pk2_ot1_exit_td_flag, 0)
        AS has_pk2_ot1_exit_td_flag

    , ISNULL(e.has_pk3_pk6_exit_td_flag, 0)
        AS has_pk3_pk6_exit_td_flag

    , ISNULL(e.has_mpl_exit_td_flag, 0)
        AS has_mpl_exit_td_flag

    , ISNULL(e.has_pns_exit_td_flag, 0)
        AS has_pns_exit_td_flag


    /* Другие срочные вклады вне окна */
    , ISNULL(ot.has_other_rub_td_flag, 0)
        AS has_other_rub_td_flag


    /* ========================================================
       НС
       ======================================================== */

    , ISNULL(bs.ns_start_sum, 0)
        AS ns_start_sum

    , CASE
          WHEN ISNULL(bs.ns_start_sum, 0) > 1000
              THEN 1
          ELSE 0
      END AS has_ns_gt_1000_flag

    , ISNULL(es.ns_end_sum, 0)
        AS ns_end_sum

    , ISNULL(es.ns_end_sum, 0)
      - ISNULL(bs.ns_start_sum, 0)
        AS ns_delta

    , CASE
          WHEN ISNULL(es.ns_end_sum, 0)
               < ISNULL(bs.ns_start_sum, 0)
              THEN 1
          ELSE 0
      END AS ns_decrease_flag


    /* ========================================================
       ОТКРЫТЫЕ ВКЛАДЫ — ОБЪЁМЫ
       ======================================================== */

    , ISNULL(o.opened_fu, 0)
        AS opened_fu

    , ISNULL(o.opened_nov, 0)
        AS opened_nov

    , ISNULL(o.opened_pr2_pr3, 0)
        AS opened_pr2_pr3

    , ISNULL(o.opened_ndp_ndm, 0)
        AS opened_ndp_ndm

    , ISNULL(o.opened_pk2_ot1, 0)
        AS opened_pk2_ot1

    , ISNULL(o.opened_pk3_pk6, 0)
        AS opened_pk3_pk6

    , ISNULL(o.opened_mpl, 0)
        AS opened_mpl

    , ISNULL(o.opened_pns, 0)
        AS opened_pns

    , ISNULL(o.opened_other, 0)
        AS opened_other

    , ISNULL(o.opened_total, 0)
        AS opened_total


    /* ========================================================
       ОТКРЫТЫЕ ВКЛАДЫ — ФЛАГИ
       ======================================================== */

    , ISNULL(o.has_opened_fu_flag, 0)
        AS has_opened_fu_flag

    , ISNULL(o.has_opened_nov_flag, 0)
        AS has_opened_nov_flag

    , ISNULL(o.has_opened_pr2_pr3_flag, 0)
        AS has_opened_pr2_pr3_flag

    , ISNULL(o.has_opened_ndp_ndm_flag, 0)
        AS has_opened_ndp_ndm_flag

    , ISNULL(o.has_opened_pk2_ot1_flag, 0)
        AS has_opened_pk2_ot1_flag

    , ISNULL(o.has_opened_pk3_pk6_flag, 0)
        AS has_opened_pk3_pk6_flag

    , ISNULL(o.has_opened_mpl_flag, 0)
        AS has_opened_mpl_flag

    , ISNULL(o.has_opened_pns_flag, 0)
        AS has_opened_pns_flag


INTO #client_mart

FROM #client_panel cp

LEFT JOIN #client_attr_flags caf
    ON caf.cli_id = cp.cli_id

LEFT JOIN base_state bs
    ON bs.period_id = cp.period_id
   AND bs.cli_id = cp.cli_id

LEFT JOIN end_state es
    ON es.period_id = cp.period_id
   AND es.cli_id = cp.cli_id

LEFT JOIN other_td_flag ot
    ON ot.period_id = cp.period_id
   AND ot.cli_id = cp.cli_id

LEFT JOIN sms_clients sms
    ON sms.period_id = cp.period_id
   AND sms.cli_id = cp.cli_id

LEFT JOIN exit_agg e
    ON e.period_id = cp.period_id
   AND e.cli_id = cp.cli_id

LEFT JOIN opened_agg o
    ON o.period_id = cp.period_id
   AND o.cli_id = cp.cli_id;


/* ============================================================
   12. ИТОГОВОЕ ПОЛОТНО ДЛЯ EXCEL
   ============================================================ */

SELECT
      cli_id

    , cohort_month
    , observation_month

    /* Новые клиентские признаки */
    , has_premium_package_flag
    , has_salary_client_flag

    , client_base_type
    , segment_flag
    , is_sms_sent
    , base_amount_flag


    /* Вклады к выходу */
    , exit_td_sum

    , exit_fu_td_sum
    , exit_nov_td_sum
    , exit_pr2_pr3_td_sum
    , exit_ndp_ndm_td_sum
    , exit_pk2_ot1_td_sum
    , exit_pk3_pk6_td_sum
    , exit_mpl_td_sum
    , exit_pns_td_sum
    , exit_other_td_sum


    /* Флаги вкладов к выходу */
    , has_fu_exit_td_flag
    , has_nov_exit_td_flag
    , has_pr2_pr3_exit_td_flag
    , has_ndp_ndm_exit_td_flag
    , has_pk2_ot1_exit_td_flag
    , has_pk3_pk6_exit_td_flag
    , has_mpl_exit_td_flag
    , has_pns_exit_td_flag

    , has_other_rub_td_flag


    /* НС */
    , ns_start_sum
    , has_ns_gt_1000_flag
    , ns_end_sum
    , ns_delta
    , ns_decrease_flag


    /* Открытые вклады */
    , opened_fu
    , opened_nov
    , opened_pr2_pr3
    , opened_ndp_ndm
    , opened_pk2_ot1
    , opened_pk3_pk6
    , opened_mpl
    , opened_pns
    , opened_other
    , opened_total


    /* Флаги открытых вкладов */
    , has_opened_fu_flag
    , has_opened_nov_flag
    , has_opened_pr2_pr3_flag
    , has_opened_ndp_ndm_flag
    , has_opened_pk2_ot1_flag
    , has_opened_pk3_pk6_flag
    , has_opened_mpl_flag
    , has_opened_pns_flag

FROM #client_mart

ORDER BY
      cohort_month
    , observation_month
    , cli_id;
