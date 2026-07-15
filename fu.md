Да, доработал именно текущую структуру: баланс выгружается только 2 раза, справочник новых денег остаётся через `promo_new_money_rate_dict`, добавлены ФУ-вклады к выходу и ФУ-открытия. Текущий скрипт уже классифицировал открытия через SMS и справочник новых денег, поэтому я расширил этот же блок классификации, не добавляя лишнюю выгрузку баланса. 

```sql
USE [ALM];
SET NOCOUNT ON;

DECLARE @BaseDate date = '2026-05-30'; -- дата баланса было
DECLARE @EndDate  date = '2026-06-20'; -- дата баланса стало

DECLARE @ExitFrom date = DATEADD(day, 1, @BaseDate);
DECLARE @ExitTo   date = @EndDate;

DECLARE @OpenFrom date = DATEADD(day, 1, @BaseDate);
DECLARE @OpenTo   date = @EndDate;

DECLARE @eps decimal(18,6) = 0.000005;

IF OBJECT_ID('tempdb..#bal_base') IS NOT NULL DROP TABLE #bal_base;
IF OBJECT_ID('tempdb..#bal_end') IS NOT NULL DROP TABLE #bal_end;
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
          WHEN NULLIF(LTRIM(RTRIM(COALESCE(t.conv, ''))), '') IS NULL THEN 'AT_THE_END'
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
          WHEN NULLIF(LTRIM(RTRIM(COALESCE(t.conv, ''))), '') IS NULL THEN 'AT_THE_END'
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
   3. Интересуют две категории клиентов

   A. Вкладчики к выходу:
      есть срочный вклад на @BaseDate с dt_close_plan в окне
      @BaseDate + 1 ... @EndDate

   B. Клиент без вкладов к выходу, но есть НС:
      есть НС на @BaseDate
      и нет срочного вклада к выходу в этом окне
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
   4. Общие расчёты и маркировки по клиентам
   ============================================================ */
WITH client_flags AS (
    /* Если у клиента есть хотя бы один вклад или НС в балансе с маркировкой ДЧБО,
       то считаем такого клиента ДЧБО */
    SELECT
          c.cli_id
        , CASE
              WHEN EXISTS (
                  SELECT 1
                  FROM #bal_base b
                  WHERE b.cli_id = c.cli_id
                    AND b.section_name IN (N'Срочные', N'Накопительный счёт')
                    AND b.TSEGMENTNAME = N'ДЧБО'
              )
              THEN N'ДЧБО'
              ELSE N'Розница'
          END AS client_segment
    FROM #client_scope c
),

/* Сумма вкладов к выходу: всего + ФУ + не ФУ */
exit_sum AS (
    SELECT
          b.cli_id
        , SUM(b.out_rub) AS exit_td_sum

        , SUM(
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
                  THEN b.out_rub
                  ELSE 0
              END
          ) AS exit_fu_td_sum

        , SUM(
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
                  THEN 0
                  ELSE b.out_rub
              END
          ) AS exit_non_fu_td_sum

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
          ) AS has_fu_exit_td_flag
    FROM #bal_base b
    INNER JOIN #client_scope c
        ON c.cli_id = b.cli_id
    WHERE
        b.section_name = N'Срочные'
        AND b.dt_close_plan >= @ExitFrom
        AND b.dt_close_plan <= @ExitTo
    GROUP BY
        b.cli_id
),

/* Есть ли у клиента вклады, не попадающие в период выхода */
other_td_flag AS (
    SELECT
          c.cli_id
        , CASE
              WHEN EXISTS (
                  SELECT 1
                  FROM #bal_base b
                  WHERE b.cli_id = c.cli_id
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

/* Клиенты, кто получал СМС за этот период */
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

opened_by_con AS (
    SELECT
          b.cli_id
        , b.con_id
        , MIN(b.dt_open) AS dt_open
        , SUM(b.out_rub) AS out_rub
        , MIN(b.rate_con) AS rate_con
        , MIN(b.termdays) AS termdays
        , CASE
              WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(b.conv_norm, ''))), '')) IS NULL THEN 'AT_THE_END'
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
          ) AS is_fu_opened_flag

    FROM #bal_end b
    INNER JOIN #client_scope c
        ON b.cli_id = c.cli_id
    WHERE
        b.section_name = N'Срочные'
        AND b.dt_open >= @OpenFrom
        AND b.dt_open <= @OpenTo
    GROUP BY
          b.cli_id
        , b.con_id
),

/* ============================================================
   Классификация открытых вкладов

   Порядок классификации:
   1) сначала вклады, открытые с СМС
   2) затем вклады маркетплейсов / ФУ
   3) затем вклады на новые деньги по справочнику
   4) затем остальные вклады
   ============================================================ */
opened_classified AS (
    SELECT
          o.cli_id
        , o.con_id
        , o.out_rub
        , CASE
              WHEN sms_promo.con_id IS NOT NULL THEN N'sms'
              WHEN o.is_fu_opened_flag = 1 THEN N'fu'
              WHEN promo_dict.id IS NOT NULL THEN N'promo'
              ELSE N'other'
          END AS open_category
    FROM opened_by_con o

    OUTER APPLY (
        SELECT TOP (1)
              v.con_id
        FROM alm.ehd.VW_promocode_contracts_short v WITH (NOLOCK)
        WHERE
            v.cli_id = o.cli_id
            AND v.con_id = o.con_id
            AND v.con_id IS NOT NULL
            AND v.messageid IS NOT NULL
            AND v.msgbegindate >= @OpenFrom
            AND v.msgbegindate < DATEADD(day, 1, @OpenTo)
            AND v.dt_open_fact >= @OpenFrom
            AND v.dt_open_fact < DATEADD(day, 1, @OpenTo)
        ORDER BY
              v.msgbegindate DESC
            , v.dt_open_fact DESC
            , v.con_id DESC
    ) sms_promo

    OUTER APPLY (
        SELECT TOP (1)
              d.id
        FROM [ALM_TEST].[WORK].[promo_new_money_rate_dict] d WITH (NOLOCK)
        WHERE
            d.is_active = 1
            AND o.dt_open BETWEEN d.date_from AND d.date_to
            AND o.termdays BETWEEN d.term_min AND d.term_max
            AND o.conv_norm = d.conv_type
            AND o.out_rub BETWEEN d.amount_from AND d.amount_to
            AND ABS(o.rate_con - d.promo_rate) <= @eps
        ORDER BY
              d.date_from DESC
            , d.id DESC
    ) promo_dict
),

opened_agg AS (
    SELECT
          cli_id
        , SUM(CASE WHEN open_category = N'sms'   THEN out_rub ELSE 0 END) AS opened_sms
        , SUM(CASE WHEN open_category = N'fu'    THEN out_rub ELSE 0 END) AS opened_fu
        , SUM(CASE WHEN open_category = N'promo' THEN out_rub ELSE 0 END) AS opened_promo
        , SUM(CASE WHEN open_category = N'other' THEN out_rub ELSE 0 END) AS opened_other
        , SUM(out_rub) AS opened_total
    FROM opened_classified
    GROUP BY
        cli_id
)

SELECT
      c.cli_id
    , c.client_base_type
    , f.client_segment AS segment_flag

    , CASE
          WHEN sms.cli_id IS NOT NULL THEN 1
          ELSE 0
      END AS is_sms_sent

    /* Единый бакет, но с разной смысловой базой */
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
    , ISNULL(e.exit_non_fu_td_sum, 0) AS exit_non_fu_td_sum

    /* 1 = есть вклад ФУ к выходу */
    , ISNULL(e.has_fu_exit_td_flag, 0) AS has_fu_exit_td_flag

    /* 0 = нет других рублёвых срочных вкладов на @BaseDate, кроме выходящих в окне
       1 = есть другие рублёвые срочные вклады на @BaseDate */
    , ISNULL(ot.has_other_rub_td_flag, 0) AS has_other_rub_td_flag

    /* Накопительные счета */
    , ISNULL(ns1.ns_start_sum, 0) AS ns_start_sum

    , CASE
          WHEN ISNULL(ns1.ns_start_sum, 0) > 1000 THEN 1
          ELSE 0
      END AS has_ns_gt_1000_flag

    , ISNULL(ns2.ns_end_sum, 0) AS ns_end_sum
    , ISNULL(ns2.ns_end_sum, 0) - ISNULL(ns1.ns_start_sum, 0) AS ns_delta

    , CASE
          WHEN ISNULL(ns2.ns_end_sum, 0) < ISNULL(ns1.ns_start_sum, 0) THEN 1
          ELSE 0
      END AS ns_decrease_flag

    /* Открытые срочные вклады за период */
    , ISNULL(o.opened_sms, 0) AS opened_sms
    , ISNULL(o.opened_fu, 0) AS opened_fu
    , ISNULL(o.opened_promo, 0) AS opened_promo
    , ISNULL(o.opened_other, 0) AS opened_other
    , ISNULL(o.opened_total, 0) AS opened_total

INTO #client_mart
FROM #client_scope c
LEFT JOIN client_flags f
    ON c.cli_id = f.cli_id
LEFT JOIN sms_clients sms
    ON c.cli_id = sms.cli_id
LEFT JOIN exit_sum e
    ON c.cli_id = e.cli_id
LEFT JOIN other_td_flag ot
    ON c.cli_id = ot.cli_id
LEFT JOIN ns_start ns1
    ON c.cli_id = ns1.cli_id
LEFT JOIN ns_end ns2
    ON c.cli_id = ns2.cli_id
LEFT JOIN opened_agg o
    ON c.cli_id = o.cli_id;


/* ============================================================
   Итоговый селект
   ============================================================ */
SELECT
      cli_id
    , client_base_type
    , segment_flag
    , is_sms_sent
    , base_amount_flag

    , exit_td_sum
    , exit_fu_td_sum
    , exit_non_fu_td_sum
    , has_fu_exit_td_flag
    , has_other_rub_td_flag

    , ns_start_sum
    , has_ns_gt_1000_flag
    , ns_end_sum
    , ns_delta
    , ns_decrease_flag

    , opened_sms
    , opened_fu
    , opened_promo
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
```
