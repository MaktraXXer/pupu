
USE [ALM_TEST];
GO

SET ANSI_NULLS ON;
GO

SET QUOTED_IDENTIFIER ON;
GO

CREATE OR ALTER FUNCTION [reports].[fn_NewAttractionVolumes_forDimaMASS_v1.6]
(
      @ReportDate date
    , @OpenFrom date
    , @OpenTo date
    , @ExcludeMP bit = 1
)
RETURNS TABLE
AS
RETURN
(
    WITH mp_list AS
    (
        SELECT DISTINCT prod_name_res
        FROM (VALUES
              (N'ДОМа надёжно')
            , (N'Всё в ДОМ')
            , (N'Надёжный')
            , (N'Надёжный VIP')
            , (N'Надёжный премиум')
            , (N'Надёжныйпромо')
            , (N'Надёжный промо')
            , (N'Надёжный старт')
            , (N'Надёжный T2')
            , (N'Надёжный Т2')
            , (N'Надёжный Мегафон')
            , (N'Надёжныйпрайм')
            , (N'Надёжный прайм')
            , (N'Надёжный процент')
            , (N'Надёжныйпроцент')
            , (N'Могучий')
        ) m(prod_name_res)
    ),

    src AS
    (
        SELECT
              v.CON_ID
            , v.CLI_ID
            , v.OUT_RUB
            , v.TSegmentName
            , v.PROD_NAME_res
            , v.termdays
            , v.rate_con
            , v.conv
            , CAST(v.dt_open AS date) AS dt_open

            , CASE
                WHEN mp.prod_name_res IS NOT NULL THEN 1
                ELSE 0
              END AS is_mp

            , CASE
                WHEN v.termdays BETWEEN 45 AND 70
                 AND ISNULL(v.conv, 'AT_THE_END') = 'AT_THE_END'
                 AND ABS(v.rate_con - 0.1500) <= 0.0005
                    THEN 1
                ELSE 0
              END AS special_2m

        FROM ALM.ALM.vw_balance_rest_all v WITH (NOLOCK)

        LEFT JOIN mp_list mp
            ON mp.prod_name_res = v.PROD_NAME_res

        WHERE v.DT_REP = @ReportDate
          AND v.dt_open >= @OpenFrom
          AND v.dt_open < DATEADD(day, 1, @OpenTo)

          AND v.section_name = N'Срочные'
          AND v.block_name = N'Привлечение ФЛ'
          AND v.ap = N'Пассив'
          AND v.od_flag = 1
          AND v.cur = '810'
          AND v.OUT_RUB > 0
          AND ISNULL(v.is_floatrate, 0) = 0
          AND v.rate_con IS NOT NULL
          AND v.termdays BETWEEN 28 AND 1830

          AND NOT EXISTS
          (
              SELECT 1
              FROM LIQUIDITY.liq.man_FloatContracts fc WITH (NOLOCK)
              WHERE fc.CON_ID = v.CON_ID
          )

          AND (
              @ExcludeMP = 0
              OR mp.prod_name_res IS NULL
          )

          AND (
              v.TSegmentName IS NULL
              OR v.TSegmentName <> N'ДЧБО'
          )

          AND (
              v.PROD_NAME_res IS NULL
              OR v.PROD_NAME_res NOT IN
              (
                  N'Классический',
                  N'Привилегия',
                  N'Достояние',
                  N'Достояние без опций',
                  N'Достояние с пополнением и снятием',
                  N'Достояние (перем.ст.) без опций',
                  N'Достояние (перем.ст.) с пополнением и снятием'
              )
          )
    ),

    special_contracts AS
    (
        SELECT DISTINCT
              CON_ID
            , CLI_ID
            , dt_open
        FROM src
        WHERE special_2m = 1
          AND is_mp = 0
          AND (
              PROD_NAME_res IS NULL
              OR PROD_NAME_res <> N'Активный процент'
          )
          AND CLI_ID IS NOT NULL
          AND CON_ID IS NOT NULL
    ),

    special_clients AS
    (
        SELECT DISTINCT CLI_ID
        FROM special_contracts
    ),

    history_contracts AS
    (
        SELECT
              h.CLI_ID
            , h.CON_ID
            , MIN(CAST(h.DT_OPEN AS date)) AS dt_open

        FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK)

        INNER JOIN special_clients sc
            ON sc.CLI_ID = h.CLI_ID

        WHERE h.DT_OPEN IS NOT NULL
          AND h.DT_OPEN < DATEADD(day, 1, @OpenTo)
          AND h.CON_ID IS NOT NULL
          AND h.cli_short_name = N'ФЛ'
          AND h.cur = 'RUR'

          AND (
              h.PROD_NAME IS NULL
              OR h.PROD_NAME NOT IN
              (
                  N'Агентские эскроу ФЛ по ставке КС+спред',
                  N'Спец. банк.счёт',
                  N'Залоговый',
                  N'Агентские эскроу ФЛ по ставке КС/2',
                  N'Эскроу',
                  N'Депозит ФЛ (суррогатный договор для счета Новой Афины)'
              )
          )

        GROUP BY h.CLI_ID, h.CON_ID
    ),

    prior_clients AS
    (
        SELECT DISTINCT CLI_ID
        FROM history_contracts
        WHERE dt_open < @OpenFrom
    ),

    existing_special AS
    (
        SELECT
              s.CON_ID
            , s.CLI_ID
            , s.dt_open

        FROM special_contracts s

        INNER JOIN prior_clients p
            ON p.CLI_ID = s.CLI_ID
    ),

    special_open AS
    (
        SELECT
              s.CON_ID
            , s.CLI_ID
            , s.dt_open
            , CAST(ISNULL(b.open_balance, 0) AS decimal(28,6))
                AS open_amount

        FROM existing_special s

        OUTER APPLY
        (
            SELECT
                SUM(CAST(b.OUT_RUB AS decimal(28,6)))
                    AS open_balance

            FROM LIQUIDITY.liq.DepositContract_Saldo b
                WITH (NOLOCK)

            WHERE b.CON_ID = s.CON_ID
              AND b.DT_FROM < DATEADD(day, 1, s.dt_open)
              AND (
                  b.DT_TO IS NULL
                  OR b.DT_TO >= s.dt_open
              )
              AND b.OUT_RUB IS NOT NULL
        ) b
    ),

    special_days AS
    (
        SELECT
              CLI_ID
            , dt_open
            , SUM(open_amount) AS total_open_amount

        FROM special_open

        GROUP BY CLI_ID, dt_open
    ),

    analysis_months AS
    (
        SELECT
              CLI_ID
            , DATEFROMPARTS(
                  YEAR(dt_open),
                  MONTH(dt_open),
                  1
              ) AS month_start
            , MAX(dt_open) AS max_open

        FROM special_days

        GROUP BY
              CLI_ID
            , YEAR(dt_open)
            , MONTH(dt_open)
    ),

    analysis_windows AS
    (
        SELECT
              CLI_ID
            , month_start
            , DATEADD(month, -1, month_start) AS prev_start
            , max_open

        FROM analysis_months
    ),

    saldo_clipped AS
    (
        SELECT
              w.CLI_ID
            , w.month_start
            , w.prev_start
            , w.max_open
            , CAST(s.OUT_RUB AS decimal(28,6)) AS out_rub
            , bounds.effective_from
            , bounds.effective_to

        FROM analysis_windows w

        INNER JOIN history_contracts h
            ON h.CLI_ID = w.CLI_ID
           AND h.dt_open <= w.max_open

        INNER JOIN LIQUIDITY.liq.DepositContract_Saldo s
            WITH (NOLOCK)
            ON s.CON_ID = h.CON_ID

           AND s.DT_FROM < DATEADD(day, 1, w.max_open)

           AND (
               s.DT_TO IS NULL
               OR s.DT_TO >= w.prev_start
           )

        CROSS APPLY
        (
            SELECT
                (
                    SELECT MAX(v.dt)
                    FROM (VALUES
                          (CAST(s.DT_FROM AS date))
                        , (h.dt_open)
                        , (w.prev_start)
                    ) v(dt)
                ) AS effective_from

                , (
                    SELECT MIN(v.dt)
                    FROM (VALUES
                          (ISNULL(CAST(s.DT_TO AS date), w.max_open))
                        , (w.max_open)
                    ) v(dt)
                ) AS effective_to
        ) bounds

        WHERE s.OUT_RUB IS NOT NULL
          AND bounds.effective_from <= bounds.effective_to
    ),

    saldo_events AS
    (
        SELECT
              s.CLI_ID
            , s.month_start
            , e.event_date
            , e.delta

        FROM saldo_clipped s

        CROSS APPLY
        (
            VALUES
            (
                s.effective_from,
                s.out_rub
            ),
            (
                CASE
                    WHEN s.effective_to < s.max_open
                        THEN DATEADD(day, 1, s.effective_to)
                    ELSE NULL
                END,
                -s.out_rub
            )
        ) e(event_date, delta)

        WHERE e.event_date IS NOT NULL

        UNION ALL

        SELECT
              w.CLI_ID
            , w.month_start
            , w.prev_start
            , CAST(0 AS decimal(28,6))

        FROM analysis_windows w

        UNION ALL

        SELECT
              w.CLI_ID
            , w.month_start
            , w.month_start
            , CAST(0 AS decimal(28,6))

        FROM analysis_windows w

        UNION ALL

        SELECT
              d.CLI_ID
            , DATEFROMPARTS(
                  YEAR(d.dt_open),
                  MONTH(d.dt_open),
                  1
              )
            , d.dt_open
            , CAST(0 AS decimal(28,6))

        FROM special_days d
    ),

    event_agg AS
    (
        SELECT
              CLI_ID
            , month_start
            , event_date
            , SUM(delta) AS delta

        FROM saldo_events

        GROUP BY
              CLI_ID
            , month_start
            , event_date
    ),

    timeline AS
    (
        SELECT
              CLI_ID
            , month_start
            , event_date

            , SUM(delta) OVER
              (
                  PARTITION BY CLI_ID, month_start
                  ORDER BY event_date
                  ROWS BETWEEN UNBOUNDED PRECEDING
                           AND CURRENT ROW
              ) AS balance

        FROM event_agg
    ),

    timeline_stats AS
    (
        SELECT
              CLI_ID
            , month_start
            , event_date
            , balance

            , MAX(
                CASE
                    WHEN event_date < month_start
                        THEN balance
                END
              ) OVER
              (
                  PARTITION BY CLI_ID, month_start
              ) AS prev_month_peak

            , MAX(
                CASE
                    WHEN event_date >= month_start
                        THEN balance
                END
              ) OVER
              (
                  PARTITION BY CLI_ID, month_start
                  ORDER BY event_date
                  ROWS BETWEEN UNBOUNDED PRECEDING
                           AND 1 PRECEDING
              ) AS current_month_peak_before

        FROM timeline
    ),

    day_balances AS
    (
        SELECT
              d.CLI_ID
            , d.dt_open
            , d.total_open_amount
            , ISNULL(t.balance, 0) AS balance_today

            , CASE
                WHEN ISNULL(t.prev_month_peak, 0)
                   >= ISNULL(t.current_month_peak_before, 0)
                    THEN ISNULL(t.prev_month_peak, 0)

                ELSE ISNULL(t.current_month_peak_before, 0)
              END AS comparison_peak

        FROM special_days d

        LEFT JOIN timeline_stats t
            ON t.CLI_ID = d.CLI_ID
           AND t.month_start = DATEFROMPARTS(
                  YEAR(d.dt_open),
                  MONTH(d.dt_open),
                  1
               )
           AND t.event_date = d.dt_open
    ),

    day_budget AS
    (
        SELECT
              CLI_ID
            , dt_open
            , total_open_amount

            , CASE
                WHEN total_open_amount <= 0 THEN 0

                WHEN balance_today <= comparison_peak THEN 0

                WHEN balance_today - comparison_peak
                     >= total_open_amount
                    THEN total_open_amount

                ELSE balance_today - comparison_peak
              END AS new_money_budget

        FROM day_balances
    ),

    special_allocation AS
    (
        SELECT
              s.CON_ID

            , CASE
                WHEN s.open_amount <= 0
                  OR d.total_open_amount <= 0
                    THEN 0

                WHEN d.new_money_budget
                     >= d.total_open_amount - 0.01
                    THEN 2

                WHEN d.new_money_budget > 0
                    THEN 1

                ELSE 0
              END AS new_money_status

            , CAST(
                CASE
                    WHEN d.total_open_amount > 0
                        THEN d.new_money_budget
                             / NULLIF(d.total_open_amount, 0)
                    ELSE 0
                END
                AS decimal(18,12)
              ) AS new_money_share

        FROM special_open s

        INNER JOIN day_budget d
            ON d.CLI_ID = s.CLI_ID
           AND d.dt_open = s.dt_open
    ),

    joined AS
    (
        SELECT
              s.*
            , d.dict_match

            , a.[Нов]
            , a.[НДП]
            , a.[НДМ]
            , a.[Мпл]
            , a.[Пнс]
            , a.[От1]
            , a.[Пк2]

            , CASE
                WHEN s.special_2m = 1
                 AND s.CLI_ID IS NOT NULL
                 AND p.CLI_ID IS NULL
                    THEN 1
                ELSE 0
              END AS is_new_client

            , ISNULL(sa.new_money_status, 0)
                AS special_new_money_status

            , ISNULL(sa.new_money_share, 0)
                AS special_new_money_share

        FROM src s

        LEFT JOIN prior_clients p
            ON p.CLI_ID = s.CLI_ID

        LEFT JOIN special_allocation sa
            ON sa.CON_ID = s.CON_ID

        OUTER APPLY
        (
            SELECT TOP (1)
                  a.[Нов]
                , a.[НДП]
                , a.[НДМ]
                , a.[Мпл]
                , a.[Пнс]
                , a.[От1]
                , a.[Пк2]

            FROM ALM.ehd.attr_DepoFLConditions a
                WITH (NOLOCK)

            WHERE a.CON_ID = s.CON_ID
              AND s.is_mp = 0

            ORDER BY
                  a.DT_UPDATE DESC
                , a.loaddate DESC
        ) a

        OUTER APPLY
        (
            SELECT TOP (1)
                1 AS dict_match

            FROM ALM_TEST.WORK.promo_new_money_rate_dict d
                WITH (NOLOCK)

            WHERE s.is_mp = 0

              AND s.dt_open
                  BETWEEN d.date_from AND d.date_to

              AND s.termdays
                  BETWEEN d.term_min AND d.term_max

              AND d.conv_type =
                  CASE
                      WHEN ISNULL(s.conv, 'AT_THE_END') = 'AT_THE_END'
                          THEN 'AT_THE_END'
                      ELSE 'NOT_AT_THE_END'
                  END

              AND s.OUT_RUB
                  BETWEEN d.amount_from AND d.amount_to

              AND ABS(s.rate_con - d.promo_rate) <= 0.0005
        ) d
    ),

    matched AS
    (
        SELECT
              j.*

            , CASE
                WHEN ISNULL(TRY_CAST(j.[Нов] AS int), 0) = 1
                    THEN 1
                ELSE 0
              END AS is_nov

            , CASE
                WHEN ISNULL(TRY_CAST(j.[НДП] AS int), 0) = 1
                  OR ISNULL(TRY_CAST(j.[НДМ] AS int), 0) = 1
                    THEN 1
                ELSE 0
              END AS is_ndp_ndm

            , CASE
                WHEN ISNULL(TRY_CAST(j.[Пнс] AS int), 0) = 1
                  OR ISNULL(TRY_CAST(j.[Мпл] AS int), 0) = 1
                    THEN 1
                ELSE 0
              END AS is_pns_mpl

            , CASE
                WHEN ISNULL(TRY_CAST(j.[От1] AS int), 0) = 1
                  OR ISNULL(TRY_CAST(j.[Пк2] AS int), 0) = 1
                    THEN 1
                ELSE 0
              END AS retention

        FROM joined j
    ),

    classified AS
    (
        SELECT
              m.CON_ID
            , m.OUT_RUB
            , m.rate_con
            , m.termdays
            , m.special_new_money_share

            , CASE

                WHEN m.is_mp = 1
                    THEN N'Маркетплейсы'

                WHEN m.PROD_NAME_res = N'Активный процент'
                    THEN N'Активный процент'

                WHEN m.special_2m = 1
                 AND m.is_new_client = 1
                    THEN N'Новые клиенты'

                WHEN m.special_2m = 1
                 AND m.special_new_money_status = 2
                    THEN N'Новые деньги'

                WHEN m.special_2m = 1
                 AND m.special_new_money_status = 1
                    THEN N'Частично новые деньги'

                WHEN m.special_2m = 1
                    THEN N'Другие вклады со ставкой новых денег'

                WHEN m.retention = 1
                    THEN N'Автоматический механизм удержания'

                WHEN m.is_nov = 1
                 AND m.dict_match = 1
                 AND m.is_pns_mpl = 0
                    THEN N'Новые клиенты'

                WHEN m.is_ndp_ndm = 1
                 AND m.dict_match = 1
                 AND m.is_pns_mpl = 0
                    THEN N'Новые деньги'

                WHEN (m.is_nov = 1 OR m.is_ndp_ndm = 1)
                 AND m.dict_match IS NULL
                 AND m.is_pns_mpl = 0
                    THEN N'Новые деньги без справочника'

                WHEN m.dict_match = 1
                 AND m.is_pns_mpl = 0
                    THEN N'Другие вклады со ставкой новых денег'

                WHEN m.is_pns_mpl = 1
                    THEN N'Вклады ПНС и МПЛ'

                ELSE N'Остальные вклады'

              END AS segment_name

            , CASE
                WHEN m.termdays BETWEEN 26 AND 44 THEN 31
                WHEN m.termdays BETWEEN 45 AND 70 THEN 61
                WHEN m.termdays BETWEEN 85 AND 119 THEN 91
                WHEN m.termdays BETWEEN 120 AND 140 THEN 124
                WHEN m.termdays BETWEEN 141 AND 170 THEN 151
                WHEN m.termdays BETWEEN 175 AND 200 THEN 181
                WHEN m.termdays BETWEEN 201 AND 230 THEN 212
                WHEN m.termdays BETWEEN 231 AND 250 THEN 243
                WHEN m.termdays BETWEEN 251 AND 290 THEN 274
                WHEN m.termdays BETWEEN 340 AND 405 THEN 365
                WHEN m.termdays BETWEEN 540 AND 621 THEN 550
                WHEN m.termdays BETWEEN 720 AND 763 THEN 750
                WHEN m.termdays BETWEEN 1090 AND 1140 THEN 1100
                WHEN m.termdays BETWEEN 1450 AND 1475 THEN 1460
                WHEN m.termdays BETWEEN 1795 AND 1830 THEN 1825
                ELSE m.termdays
              END AS bucket_code

        FROM matched m
    ),

    agg AS
    (
        SELECT
              bucket_code

            , CASE
                WHEN GROUPING(segment_name) = 1
                    THEN N'Итого'
                ELSE segment_name
              END AS segment_name

            , GROUPING(segment_name) AS is_total

            , SUM(CAST(OUT_RUB AS decimal(24,2)))
                AS segment_vol

            , SUM(
                CAST(OUT_RUB AS decimal(24,2))
                * CAST(rate_con AS decimal(12,8))
              ) AS weighted_rate_sum

            , SUM(
                CASE
                    WHEN segment_name = N'Частично новые деньги'
                        THEN
                            CAST(OUT_RUB AS decimal(24,2))
                            * special_new_money_share
                    ELSE 0
                END
              ) AS partial_new_money_vol

        FROM classified

        GROUP BY GROUPING SETS
        (
              (bucket_code, segment_name)
            , (bucket_code)
        )
    ),

    totals AS
    (
        SELECT SUM(segment_vol) AS grand_total
        FROM agg
        WHERE is_total = 0
    )

    SELECT
          a.bucket_code AS Bucket
        , a.segment_name AS Segment

        , CAST(
            a.segment_vol / 1000000000.0
            AS decimal(18,6)
          ) AS SegmentVolume

        , CAST(
            a.segment_vol * 1.0
            / NULLIF(t.grand_total, 0)
            AS decimal(18,4)
          ) AS SegmentSharePct

        , CAST(
            a.weighted_rate_sum
            / NULLIF(a.segment_vol, 0)
            AS decimal(9,6)
          ) AS ClientRate

        , CAST(
            a.partial_new_money_vol / 1000000000.0
            AS decimal(18,6)
          ) AS PartialNewMoneyVolume

    FROM agg a
    CROSS JOIN totals t
);
GO





SELECT *
FROM ALM_TEST.reports.fn_NewAttractionVolumes_forDimaMASS_v1.6(
    '2026-11-04',
    '2026-10-20',
    '2026-11-04',
    0
)
ORDER BY Bucket, Segment;
