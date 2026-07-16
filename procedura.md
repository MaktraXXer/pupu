USE [ALM];
SET NOCOUNT ON;

DECLARE @BaseDate date = '2026-05-31'; -- дата баланса было
DECLARE @EndDate  date = '2026-06-30'; -- дата баланса стало

DECLARE @ExitFrom date = DATEADD(day, 1, @BaseDate);
DECLARE @ExitTo   date = @EndDate;

DECLARE @OpenFrom date = DATEADD(day, 1, @BaseDate);
DECLARE @OpenTo   date = @EndDate;

IF OBJECT_ID('tempdb..#bal_base') IS NOT NULL DROP TABLE #bal_base;
IF OBJECT_ID('tempdb..#bal_end') IS NOT NULL DROP TABLE #bal_end;
IF OBJECT_ID('tempdb..#attr_flags') IS NOT NULL DROP TABLE #attr_flags;
IF OBJECT_ID('tempdb..#client_scope') IS NOT NULL DROP TABLE #client_scope;
IF OBJECT_ID('tempdb..#client_mart') IS NOT NULL DROP TABLE #client_mart;


/* ============================================================
   1. Баланс на дату было
   ============================================================ */
SELECT
      CAST(t.dt_rep AS date) AS dt_rep
    , CAST(t.cli_id AS bigint) AS cli_id
    , CAST(t.con_id AS bigint) AS con_id
    , CAST(t.dt_open AS date) AS dt_open
    , CAST(t.dt_close_plan AS date) AS dt_close_plan
    , t.section_name
    , t.PROD_NAME_res
    , CAST(t.out_rub AS decimal(38,6)) AS out_rub
    , CAST(t.rate_con AS decimal(18,6)) AS rate_con
    , CASE
          WHEN NULLIF(LTRIM(RTRIM(COALESCE(t.conv, ''))), '') IS NULL
              THEN 'AT_THE_END'
          ELSE UPPER(LTRIM(RTRIM(t.conv)))
      END AS conv_norm
    , CAST(t.termdays AS int) AS termdays
    , t.TSEGMENTNAME
INTO #bal_base
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE
    t.dt_rep = @BaseDate
    AND t.section_name IN (N'Срочные', N'Накопительный счёт')
    AND t.block_name = N'Привлечение ФЛ'
    AND t.acc_role   = N'LIAB'
    AND t.od_flag    = 1
    AND t.cur        = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;


/* ============================================================
   2. Баланс на дату стало
   ============================================================ */
SELECT
      CAST(t.dt_rep AS date) AS dt_rep
    , CAST(t.cli_id AS bigint) AS cli_id
    , CAST(t.con_id AS bigint) AS con_id
    , CAST(t.dt_open AS date) AS dt_open
    , CAST(t.dt_close_plan AS date) AS dt_close_plan
    , t.section_name
    , t.PROD_NAME_res
    , CAST(t.out_rub AS decimal(38,6)) AS out_rub
    , CAST(t.rate_con AS decimal(18,6)) AS rate_con
    , CASE
          WHEN NULLIF(LTRIM(RTRIM(COALESCE(t.conv, ''))), '') IS NULL
              THEN 'AT_THE_END'
          ELSE UPPER(LTRIM(RTRIM(t.conv)))
      END AS conv_norm
    , CAST(t.termdays AS int) AS termdays
    , t.TSEGMENTNAME
INTO #bal_end
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE
    t.dt_rep = @EndDate
    AND t.section_name IN (N'Срочные', N'Накопительный счёт')
    AND t.block_name = N'Привлечение ФЛ'
    AND t.acc_role   = N'LIAB'
    AND t.od_flag    = 1
    AND t.cur        = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;


/* ============================================================
   3. Последние актуальные признаки договора

   Таблица attr_DepoFLConditions читается один раз.

   По каждому con_id берём последнюю запись:
   1) по DT_UPDATE;
   2) затем по loaddate.

   Используемые признаки:
   - Мпл;
   - Пк2;
   - Нов;
   - НДП;
   - НДМ.
   ============================================================ */
WITH relevant_con_id AS (
    SELECT con_id
    FROM #bal_base
    WHERE con_id IS NOT NULL

    UNION

    SELECT con_id
    FROM #bal_end
    WHERE con_id IS NOT NULL
),

attr_ranked AS (
    SELECT
          CAST(a.CON_ID AS bigint) AS con_id

        , CASE
              WHEN ISNULL(TRY_CAST(a.[Мпл] AS int), 0) = 1 THEN 1
              ELSE 0
          END AS is_mpl_flag

        , CASE
              WHEN ISNULL(TRY_CAST(a.[Пк2] AS int), 0) = 1 THEN 1
              ELSE 0
          END AS is_pk2_flag

        , CASE
              WHEN ISNULL(TRY_CAST(a.[Нов] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[НДП] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[НДМ] AS int), 0) = 1
              THEN 1
              ELSE 0
          END AS is_new_money_flag

        , ROW_NUMBER() OVER (
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
    , is_mpl_flag
    , is_pk2_flag
    , is_new_money_flag
INTO #attr_flags
FROM attr_ranked
WHERE rn = 1;


/* ============================================================
   4. Интересуют две категории клиентов

   A. Вкладчики к выходу:
      есть срочный вклад на @BaseDate с dt_close_plan в окне
      @BaseDate + 1 ... @EndDate.

   B. Клиенты с НС без вкладов к выходу:
      есть НС на @BaseDate
      и нет срочного вклада к выходу в этом окне.
   ============================================================ */
SELECT
      x.cli_id
    , x.client_base_type
INTO #client_scope
FROM (
    /* A. Клиенты со срочными вкладами к выходу */
    SELECT DISTINCT
          b.cli_id
        , CAST(N'01. Вкладчики к выходу' AS nvarchar(100)) AS client_base_type
    FROM #bal_base b
    WHERE
        b.section_name = N'Срочные'
        AND b.dt_close_plan >= @ExitFrom
        AND b.dt_close_plan <= @ExitTo

    UNION ALL

    /* B. Клиенты с НС, но без срочных вкладов к выходу */
    SELECT
          ns.cli_id
        , CAST(N'02. НС без вкладов к выходу' AS nvarchar(100)) AS client_base_type
    FROM #bal_base ns
    WHERE
        ns.section_name = N'Накопительный счёт'
        AND NOT EXISTS (
            SELECT 1
            FROM #bal_base td
            WHERE
                td.cli_id = ns.cli_id
                AND td.section_name = N'Срочные'
                AND td.dt_close_plan >= @ExitFrom
                AND td.dt_close_plan <= @ExitTo
        )
    GROUP BY
        ns.cli_id
) x;


/* ============================================================
   5. Общие расчёты и маркировки по клиентам
   ============================================================ */
WITH client_flags AS (
    /* Если хотя бы один вклад или НС клиента имеет сегмент ДЧБО,
       клиент целиком считается клиентом ДЧБО */
    SELECT
          c.cli_id
        , CASE
              WHEN EXISTS (
                  SELECT 1
                  FROM #bal_base b
                  WHERE
                      b.cli_id = c.cli_id
                      AND b.section_name IN (
                            N'Срочные'
                          , N'Накопительный счёт'
                      )
                      AND b.TSEGMENTNAME = N'ДЧБО'
              )
              THEN N'ДЧБО'
              ELSE N'Розница'
          END AS client_segment
    FROM #client_scope c
),


/* ============================================================
   Вклады к выходу на уровне договора

   Каждый con_id сначала схлопывается до одной строки.

   Приоритет классификации:
   1. ФУ;
   2. МПЛ;
   3. Пк2;
   4. Нов / НДП / НДМ;
   5. остальные.
   ============================================================ */
exit_by_con AS (
    SELECT
          b.cli_id
        , b.con_id
        , SUM(b.out_rub) AS out_rub

        , MAX(
              CASE
                  WHEN b.PROD_NAME_res IN (
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

        , MAX(ISNULL(a.is_mpl_flag, 0)) AS is_mpl_flag
        , MAX(ISNULL(a.is_pk2_flag, 0)) AS is_pk2_flag
        , MAX(ISNULL(a.is_new_money_flag, 0)) AS is_new_money_flag

    FROM #bal_base b
    INNER JOIN #client_scope c
        ON c.cli_id = b.cli_id
    LEFT JOIN #attr_flags a
        ON a.con_id = b.con_id
    WHERE
        b.section_name = N'Срочные'
        AND b.dt_close_plan >= @ExitFrom
        AND b.dt_close_plan <= @ExitTo
    GROUP BY
          b.cli_id
        , b.con_id
),

exit_classified AS (
    SELECT
          e.cli_id
        , e.con_id
        , e.out_rub
        , CASE
              WHEN e.is_fu_flag = 1
                  THEN N'fu'

              WHEN e.is_mpl_flag = 1
                  THEN N'mpl'

              WHEN e.is_pk2_flag = 1
                  THEN N'pk2'

              WHEN e.is_new_money_flag = 1
                  THEN N'new_money'

              ELSE N'other'
          END AS exit_category
    FROM exit_by_con e
),

exit_sum AS (
    SELECT
          cli_id

        , SUM(out_rub) AS exit_td_sum

        , SUM(
              CASE
                  WHEN exit_category = N'fu'
                      THEN out_rub
                  ELSE 0
              END
          ) AS exit_fu_td_sum

        , SUM(
              CASE
                  WHEN exit_category = N'mpl'
                      THEN out_rub
                  ELSE 0
              END
          ) AS exit_mpl_td_sum

        , SUM(
              CASE
                  WHEN exit_category = N'pk2'
                      THEN out_rub
                  ELSE 0
              END
          ) AS exit_pk2_td_sum

        , SUM(
              CASE
                  WHEN exit_category = N'new_money'
                      THEN out_rub
                  ELSE 0
              END
          ) AS exit_new_money_td_sum

        , SUM(
              CASE
                  WHEN exit_category = N'other'
                      THEN out_rub
                  ELSE 0
              END
          ) AS exit_other_td_sum

        , MAX(
              CASE
                  WHEN exit_category = N'fu'
                      THEN 1
                  ELSE 0
              END
          ) AS has_fu_exit_td_flag

    FROM exit_classified
    GROUP BY
        cli_id
),


/* Есть ли у клиента другие срочные вклады,
   не попадающие в анализируемый период выхода */
other_td_flag AS (
    SELECT
          c.cli_id
        , CASE
              WHEN EXISTS (
                  SELECT 1
                  FROM #bal_base b
                  WHERE
                      b.cli_id = c.cli_id
                      AND b.section_name = N'Срочные'
                      AND NOT (
                              b.dt_close_plan >= @ExitFrom
                          AND b.dt_close_plan <= @ExitTo
                      )
              )
              THEN 1
              ELSE 0
          END AS has_other_rub_td_flag
    FROM #client_scope c
),


/* Остаток на НС на начало */
ns_start AS (
    SELECT
          b.cli_id
        , SUM(b.out_rub) AS ns_start_sum
    FROM #bal_base b
    INNER JOIN #client_scope c
        ON c.cli_id = b.cli_id
    WHERE
        b.section_name = N'Накопительный счёт'
    GROUP BY
        b.cli_id
),


/* Остаток на НС на конец */
ns_end AS (
    SELECT
          e.cli_id
        , SUM(e.out_rub) AS ns_end_sum
    FROM #bal_end e
    INNER JOIN #client_scope c
        ON c.cli_id = e.cli_id
    WHERE
        e.section_name = N'Накопительный счёт'
    GROUP BY
        e.cli_id
),


/* Клиенты, которые получали SMS за анализируемый период.
   Флаг клиента сохраняется независимо от классификации вкладов. */
sms_clients AS (
    SELECT DISTINCT
          CAST(m.cli_id AS bigint) AS cli_id
    FROM alm.ehd.ODS_058_VI_MESSAGE2DWH m WITH (NOLOCK)
    INNER JOIN #client_scope c
        ON c.cli_id = CAST(m.cli_id AS bigint)
    WHERE
        m.msgbegindate >= @OpenFrom
        AND m.msgbegindate < DATEADD(day, 1, @OpenTo)
        AND m.cli_id IS NOT NULL
),


/* ============================================================
   Новые открытия на уровне договора

   Каждый con_id сначала схлопывается до одной строки.

   Приоритет классификации:
   1. ФУ;
   2. МПЛ;
   3. Пк2;
   4. Нов / НДП / НДМ;
   5. остальные.
   ============================================================ */
opened_by_con AS (
    SELECT
          b.cli_id
        , b.con_id
        , MIN(b.dt_open) AS dt_open
        , SUM(b.out_rub) AS out_rub
        , MIN(b.rate_con) AS rate_con
        , MIN(b.termdays) AS termdays
        , CASE
              WHEN MIN(
                       NULLIF(
                           LTRIM(RTRIM(COALESCE(b.conv_norm, '')))
                         , ''
                       )
                   ) IS NULL
                  THEN 'AT_THE_END'
              ELSE MIN(b.conv_norm)
          END AS conv_norm

        , MAX(
              CASE
                  WHEN b.PROD_NAME_res IN (
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

        , MAX(ISNULL(a.is_mpl_flag, 0)) AS is_mpl_flag
        , MAX(ISNULL(a.is_pk2_flag, 0)) AS is_pk2_flag
        , MAX(ISNULL(a.is_new_money_flag, 0)) AS is_new_money_flag

    FROM #bal_end b
    INNER JOIN #client_scope c
        ON c.cli_id = b.cli_id
    LEFT JOIN #attr_flags a
        ON a.con_id = b.con_id
    WHERE
        b.section_name = N'Срочные'
        AND b.dt_open >= @OpenFrom
        AND b.dt_open <= @OpenTo
    GROUP BY
          b.cli_id
        , b.con_id
),

opened_classified AS (
    SELECT
          o.cli_id
        , o.con_id
        , o.out_rub
        , CASE
              WHEN o.is_fu_flag = 1
                  THEN N'fu'

              WHEN o.is_mpl_flag = 1
                  THEN N'mpl'

              WHEN o.is_pk2_flag = 1
                  THEN N'pk2'

              WHEN o.is_new_money_flag = 1
                  THEN N'new_money'

              ELSE N'other'
          END AS open_category
    FROM opened_by_con o
),

opened_agg AS (
    SELECT
          cli_id

        , SUM(
              CASE
                  WHEN open_category = N'fu'
                      THEN out_rub
                  ELSE 0
              END
          ) AS opened_fu

        , SUM(
              CASE
                  WHEN open_category = N'mpl'
                      THEN out_rub
                  ELSE 0
              END
          ) AS opened_mpl

        , SUM(
              CASE
                  WHEN open_category = N'pk2'
                      THEN out_rub
                  ELSE 0
              END
          ) AS opened_pk2

        , SUM(
              CASE
                  WHEN open_category = N'new_money'
                      THEN out_rub
                  ELSE 0
              END
          ) AS opened_new_money

        , SUM(
              CASE
                  WHEN open_category = N'other'
                      THEN out_rub
                  ELSE 0
              END
          ) AS opened_other

        , SUM(out_rub) AS opened_total

    FROM opened_classified
    GROUP BY
        cli_id
)


/* ============================================================
   6. Поклиентная витрина
   ============================================================ */
SELECT
      c.cli_id
    , c.client_base_type
    , f.client_segment AS segment_flag

    /* Клиент получал SMS в анализируемый период */
    , CASE
          WHEN sms.cli_id IS NOT NULL THEN 1
          ELSE 0
      END AS is_sms_sent

    /* Единый бакет с разной базой:
       - вкладчики: объём вкладов к выходу;
       - НС-группа: стартовый остаток НС */
    , CASE
          WHEN c.client_base_type = N'01. Вкладчики к выходу'
               AND ISNULL(e.exit_td_sum, 0) < 1500000
              THEN N'1. Выход|НС < 1.5 млн'

          WHEN c.client_base_type = N'01. Вкладчики к выходу'
               AND ISNULL(e.exit_td_sum, 0) < 5000000
              THEN N'2. Выход|НС 1.5-5 млн'

          WHEN c.client_base_type = N'01. Вкладчики к выходу'
               AND ISNULL(e.exit_td_sum, 0) >= 5000000
              THEN N'3. Выход|НС >= 5 млн'

          WHEN c.client_base_type = N'02. НС без вкладов к выходу'
               AND ISNULL(ns1.ns_start_sum, 0) < 1500000
              THEN N'1. Выход|НС < 1.5 млн'

          WHEN c.client_base_type = N'02. НС без вкладов к выходу'
               AND ISNULL(ns1.ns_start_sum, 0) < 5000000
              THEN N'2. Выход|НС 1.5-5 млн'

          WHEN c.client_base_type = N'02. НС без вкладов к выходу'
               AND ISNULL(ns1.ns_start_sum, 0) >= 5000000
              THEN N'3. Выход|НС >= 5 млн'

          ELSE N'Не определено'
      END AS base_amount_flag

    /* Вклады к выходу */
    , ISNULL(e.exit_td_sum, 0) AS exit_td_sum

    , ISNULL(e.exit_fu_td_sum, 0) AS exit_fu_td_sum
    , ISNULL(e.exit_mpl_td_sum, 0) AS exit_mpl_td_sum
    , ISNULL(e.exit_pk2_td_sum, 0) AS exit_pk2_td_sum
    , ISNULL(e.exit_new_money_td_sum, 0) AS exit_new_money_td_sum
    , ISNULL(e.exit_other_td_sum, 0) AS exit_other_td_sum

    /* 1 = у клиента есть вклад ФУ к выходу */
    , ISNULL(e.has_fu_exit_td_flag, 0) AS has_fu_exit_td_flag

    /* 1 = у клиента есть другие срочные вклады,
       не попадающие в период выхода */
    , ISNULL(ot.has_other_rub_td_flag, 0) AS has_other_rub_td_flag

    /* Накопительные счета */
    , ISNULL(ns1.ns_start_sum, 0) AS ns_start_sum

    , CASE
          WHEN ISNULL(ns1.ns_start_sum, 0) > 1000 THEN 1
          ELSE 0
      END AS has_ns_gt_1000_flag

    , ISNULL(ns2.ns_end_sum, 0) AS ns_end_sum

    , ISNULL(ns2.ns_end_sum, 0)
      - ISNULL(ns1.ns_start_sum, 0) AS ns_delta

    , CASE
          WHEN ISNULL(ns2.ns_end_sum, 0)
               < ISNULL(ns1.ns_start_sum, 0)
              THEN 1
          ELSE 0
      END AS ns_decrease_flag

    /* Открытые срочные вклады */
    , ISNULL(o.opened_fu, 0) AS opened_fu
    , ISNULL(o.opened_mpl, 0) AS opened_mpl
    , ISNULL(o.opened_pk2, 0) AS opened_pk2
    , ISNULL(o.opened_new_money, 0) AS opened_new_money
    , ISNULL(o.opened_other, 0) AS opened_other
    , ISNULL(o.opened_total, 0) AS opened_total

INTO #client_mart
FROM #client_scope c
LEFT JOIN client_flags f
    ON f.cli_id = c.cli_id
LEFT JOIN sms_clients sms
    ON sms.cli_id = c.cli_id
LEFT JOIN exit_sum e
    ON e.cli_id = c.cli_id
LEFT JOIN other_td_flag ot
    ON ot.cli_id = c.cli_id
LEFT JOIN ns_start ns1
    ON ns1.cli_id = c.cli_id
LEFT JOIN ns_end ns2
    ON ns2.cli_id = c.cli_id
LEFT JOIN opened_agg o
    ON o.cli_id = c.cli_id;


/* ============================================================
   7. Итоговый результат
   ============================================================ */
SELECT
      cli_id
    , client_base_type
    , segment_flag
    , is_sms_sent
    , base_amount_flag

    , exit_td_sum
    , exit_fu_td_sum
    , exit_mpl_td_sum
    , exit_pk2_td_sum
    , exit_new_money_td_sum
    , exit_other_td_sum

    , has_fu_exit_td_flag
    , has_other_rub_td_flag

    , ns_start_sum
    , has_ns_gt_1000_flag
    , ns_end_sum
    , ns_delta
    , ns_decrease_flag

    , opened_fu
    , opened_mpl
    , opened_pk2
    , opened_new_money
    , opened_other
    , opened_total
FROM #client_mart
ORDER BY
      client_base_type
    , segment_flag
    , is_sms_sent DESC
    , base_amount_flag
    , has_fu_exit_td_flag DESC
    , has_other_rub_td_flag DESC
    , ns_decrease_flag DESC
    , opened_total DESC
    , ns_delta ASC
    , cli_id;
