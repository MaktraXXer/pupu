
USE [ALM_TEST];
GO

SET ANSI_NULLS ON;
GO

SET QUOTED_IDENTIFIER ON;
GO

CREATE OR ALTER FUNCTION [reports].[fn_NewAttractionVolumes_forDimaPRIVATE_v1]
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
                WHEN ISNULL(v.is_floatrate, 0) = 1
                  OR v.PROD_NAME_res LIKE N'Достояние (перем.ст.)%'
                  OR EXISTS
                  (
                      SELECT 1
                      FROM LIQUIDITY.liq.man_FloatContracts fc
                           WITH (NOLOCK)
                      WHERE fc.CON_ID = v.CON_ID
                  )
                    THEN 1
                ELSE 0
              END AS is_float

            , CASE
                WHEN v.termdays BETWEEN 45 AND 70
                 AND ISNULL(v.conv, 'AT_THE_END') = 'AT_THE_END'
                 AND ABS(v.rate_con - 0.1500) <= 0.0005
                    THEN 1
                ELSE 0
              END AS special_15

            , CASE
                WHEN v.termdays BETWEEN 26 AND 44 THEN 31
                WHEN v.termdays BETWEEN 45 AND 70 THEN 61
                WHEN v.termdays BETWEEN 85 AND 119 THEN 91
                WHEN v.termdays BETWEEN 120 AND 140 THEN 122
                WHEN v.termdays BETWEEN 141 AND 170 THEN 151
                WHEN v.termdays BETWEEN 175 AND 200 THEN 181
                WHEN v.termdays BETWEEN 201 AND 230 THEN 212
                WHEN v.termdays BETWEEN 231 AND 250 THEN 243
                WHEN v.termdays BETWEEN 251 AND 290 THEN 274
                WHEN v.termdays BETWEEN 340 AND 405 THEN 367
                WHEN v.termdays BETWEEN 540 AND 621 THEN 550
                WHEN v.termdays BETWEEN 720 AND 763 THEN 730
                WHEN v.termdays BETWEEN 1090 AND 1140 THEN 1100
                WHEN v.termdays BETWEEN 1450 AND 1475 THEN 1460
                WHEN v.termdays BETWEEN 1795 AND 1830 THEN 1825
                ELSE v.termdays
              END AS base_term

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
          AND v.termdays BETWEEN 28 AND 1830

          AND (
              v.TSegmentName = N'ДЧБО'
              OR v.PROD_NAME_res IN
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

          AND (
              @ExcludeMP = 0
              OR mp.prod_name_res IS NULL
          )
    ),

    joined AS
    (
        SELECT
              s.*

            , CASE
                WHEN d.term_bucket IS NOT NULL THEN 1
                ELSE 0
              END AS dict_match

            , CASE
                WHEN s.special_15 = 1
                  OR d.term_bucket IS NOT NULL
                    THEN 1
                ELSE 0
              END AS is_promo

            , CASE
                WHEN b.base_rate_raw > 1
                    THEN b.base_rate_raw / 100.0
                ELSE b.base_rate_raw
              END AS base_rate

        FROM src s

        OUTER APPLY
        (
            SELECT TOP (1)
                d.term_bucket

            FROM ALM_TEST.WORK.promo_new_money_rate_dict d
                 WITH (NOLOCK)

            WHERE s.is_float = 0

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

            ORDER BY d.id DESC
        ) d

        OUTER APPLY
        (
            SELECT TOP (1)
                TRY_CONVERT(decimal(18,8), b.[rate])
                    AS base_rate_raw

            FROM ALM_TEST.alm_history.base_rates b
                 WITH (NOLOCK)

            WHERE s.is_float = 0

              AND b.[customer segment] = 'private'

              AND TRY_CONVERT(int, b.[term]) = s.base_term

              AND b.[convention] =
                  CASE
                      WHEN ISNULL(s.conv, 'AT_THE_END') = 'AT_THE_END'
                          THEN 'at the end'
                      ELSE 'every month'
                  END

              AND b.start_date <= s.dt_open

              AND (
                  b.end_date IS NULL
                  OR b.end_date >= s.dt_open
              )

            ORDER BY b.start_date DESC
        ) b
    ),

    promo_contracts AS
    (
        SELECT DISTINCT
              CON_ID
            , CLI_ID
            , dt_open

        FROM joined

        WHERE is_float = 0
          AND is_promo = 1
          AND CLI_ID IS NOT NULL
          AND CON_ID IS NOT NULL
    ),

    promo_clients AS
    (
        SELECT DISTINCT CLI_ID
        FROM promo_contracts
    ),

    history_contracts AS
    (
        SELECT
              h.CLI_ID
            , h.CON_ID
            , MIN(CAST(h.DT_OPEN AS date)) AS dt_open

        FROM LIQUIDITY.liq.DepositContract_all h
             WITH (NOLOCK)

        INNER JOIN promo_clients pc
            ON pc.CLI_ID = h.CLI_ID

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

    existing_promo AS
    (
        SELECT
              p.CON_ID
            , p.CLI_ID
            , p.dt_open

        FROM promo_contracts p

        INNER JOIN prior_clients pc
            ON pc.CLI_ID = p.CLI_ID
    ),

    promo_open AS
    (
        SELECT
              p.CON_ID
            , p.CLI_ID
            , p.dt_open

            , CAST(
                ISNULL(s.open_balance, 0)
                AS decimal(28,6)
              ) AS open_amount

        FROM existing_promo p

        OUTER APPLY
        (
            SELECT
                SUM(CAST(s.OUT_RUB AS decimal(28,6)))
                    AS open_balance

            FROM LIQUIDITY.liq.DepositContract_Saldo s
                 WITH (NOLOCK)

            WHERE s.CON_ID = p.CON_ID

              AND s.DT_FROM < DATEADD(day, 1, p.dt_open)

              AND (
                  s.DT_TO IS NULL
                  OR s.DT_TO >= p.dt_open
              )

              AND s.OUT_RUB IS NOT NULL
        ) s
    ),

    promo_days AS
    (
        SELECT
              CLI_ID
            , dt_open
            , SUM(open_amount) AS total_open_amount

        FROM promo_open

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

        FROM promo_days

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

        FROM promo_days d
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

        FROM promo_days d

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

    promo_allocation AS
    (
        SELECT
              p.CON_ID

            , CASE
                WHEN p.open_amount <= 0
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

        FROM promo_open p

        INNER JOIN day_budget d
            ON d.CLI_ID = p.CLI_ID
           AND d.dt_open = p.dt_open
    ),

    classified AS
    (
        SELECT
              j.CON_ID
            , j.OUT_RUB
            , j.rate_con
            , j.termdays
            , j.base_term
            , j.base_rate

            , ISNULL(a.new_money_share, 0)
                AS new_money_share

            , CASE
                WHEN j.is_float = 1
                    THEN N'Плавающая ставка'

                WHEN j.is_promo = 1
                 AND j.CLI_ID IS NOT NULL
                 AND pc.CLI_ID IS NULL
                    THEN N'Новые клиенты'

                WHEN j.is_promo = 1
                 AND a.new_money_status = 2
                    THEN N'Новые деньги'

                WHEN j.is_promo = 1
                 AND a.new_money_status = 1
                    THEN N'Частично новые деньги'

                WHEN j.is_promo = 1
                    THEN N'Промоставка на действующие деньги'

                WHEN j.rate_con > j.base_rate + 0.0000005
                    THEN N'Выше базовой ставки'

                ELSE N'Базовая ставка или ниже'
              END AS segment_name

            , CASE
                WHEN j.termdays BETWEEN 26 AND 44 THEN 31
                WHEN j.termdays BETWEEN 45 AND 70 THEN 61
                WHEN j.termdays BETWEEN 85 AND 119 THEN 91
                WHEN j.termdays BETWEEN 120 AND 140 THEN 124
                WHEN j.termdays BETWEEN 141 AND 170 THEN 151
                WHEN j.termdays BETWEEN 175 AND 200 THEN 181
                WHEN j.termdays BETWEEN 201 AND 230 THEN 212
                WHEN j.termdays BETWEEN 231 AND 250 THEN 243
                WHEN j.termdays BETWEEN 251 AND 290 THEN 274
                WHEN j.termdays BETWEEN 340 AND 405 THEN 365
                WHEN j.termdays BETWEEN 540 AND 621 THEN 550
                WHEN j.termdays BETWEEN 720 AND 763 THEN 750
                WHEN j.termdays BETWEEN 1090 AND 1140 THEN 1100
                WHEN j.termdays BETWEEN 1450 AND 1475 THEN 1460
                WHEN j.termdays BETWEEN 1795 AND 1830 THEN 1825
                ELSE j.termdays
              END AS bucket_code

        FROM joined j

        LEFT JOIN prior_clients pc
            ON pc.CLI_ID = j.CLI_ID

        LEFT JOIN promo_allocation a
            ON a.CON_ID = j.CON_ID
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
                CASE
                    WHEN rate_con IS NOT NULL
                        THEN CAST(OUT_RUB AS decimal(24,2))
                             * CAST(rate_con AS decimal(12,8))
                END
              ) AS weighted_rate_sum

            , SUM(
                CASE
                    WHEN rate_con IS NOT NULL
                        THEN CAST(OUT_RUB AS decimal(24,2))
                    ELSE 0
                END
              ) AS rate_volume

            , SUM(
                CASE
                    WHEN segment_name = N'Частично новые деньги'
                        THEN CAST(OUT_RUB AS decimal(24,2))
                             * new_money_share
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
            / NULLIF(a.rate_volume, 0)
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
FROM ALM_TEST.reports.fn_NewAttractionVolumes_forDimaPRIVATE_v1(
    '2026-11-04',
    '2026-10-20',
    '2026-11-04',
    1
)
ORDER BY Bucket, Segment;


