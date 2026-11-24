USE ALM_TEST;
SET NOCOUNT ON;

;WITH agg AS (

    SELECT
        target_flag,

        SUM(incoming_sr) AS in_sr,
        SUM(outgoing_sr) AS out_sr,

        SUM(incoming_ns) AS in_ns,
        SUM(outgoing_ns) AS out_ns,

        SUM(incoming_dvs) AS in_dvs,
        SUM(outgoing_dvs) AS out_dvs,

        SUM(incoming_other) AS in_other,
        SUM(outgoing_other) AS out_other,

        SUM(incoming_unknown) AS in_unknown,
        SUM(outgoing_unknown) AS out_unknown

    FROM WORK.RK_Oct2026_PaymentAgg

    GROUP BY target_flag

)

SELECT
    a.target_flag,
    v.raspoznano_type,

    CAST(v.incoming / 1e9 AS decimal(20,4)) AS incoming,
    CAST(v.outgoing / 1e9 AS decimal(20,4)) AS outgoing,

    CAST((v.incoming + v.outgoing) / 1e9
        AS decimal(20,4)) AS saldo,

    CAST((ABS(v.incoming) + ABS(v.outgoing)) / 1e9
        AS decimal(20,4)) AS turnover,

    CAST(
        100.0 * (ABS(v.incoming) + ABS(v.outgoing))
        / NULLIF(
            SUM(ABS(v.incoming) + ABS(v.outgoing))
            OVER (PARTITION BY v.raspoznano_type), 0
        )
        AS decimal(10,2)
    ) AS share_pct

FROM agg a

CROSS APPLY (
    VALUES
        (N'Вклады', a.in_sr, a.out_sr),
        (N'НС', a.in_ns, a.out_ns),
        (N'ДВС', a.in_dvs, a.out_dvs),
        (N'Другие распознанные', a.in_other, a.out_other),
        (N'Нераспознанные', a.in_unknown, a.out_unknown)
) v(raspoznano_type, incoming, outgoing)

ORDER BY
    v.raspoznano_type,
    a.target_flag

OPTION (MAXDOP 1, RECOMPILE);



USE ALM;
SET NOCOUNT ON;

DECLARE @DateFrom date = '2026-09-30';
DECLARE @DateTo date = '2026-11-04';

;WITH balances AS (

    SELECT
        c.client_group,

        SUM(CASE WHEN t.dt_rep = @DateFrom
                  AND t.section_name = N'Срочные'
                 THEN t.out_rub ELSE 0 END) AS depo_before,

        SUM(CASE WHEN t.dt_rep = @DateTo
                  AND t.section_name = N'Срочные'
                 THEN t.out_rub ELSE 0 END) AS depo_after,

        SUM(CASE WHEN t.dt_rep = @DateFrom
                  AND t.section_name = N'Накопительный счёт'
                 THEN t.out_rub ELSE 0 END) AS ns_before,

        SUM(CASE WHEN t.dt_rep = @DateTo
                  AND t.section_name = N'Накопительный счёт'
                 THEN t.out_rub ELSE 0 END) AS ns_after

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    INNER JOIN ALM_TEST.WORK.RK_Oct2026_ClientRegistry c
        ON c.cli_id = t.cli_id

    WHERE t.dt_rep IN (@DateFrom, @DateTo)

      AND c.client_group IN (
          N'новый клиент масс',
          N'старый клиент масс кто принес новые деньги',
          N'старый клиент масс кто не принес новые деньги'
      )

      AND t.block_name = N'Привлечение ФЛ'
      AND t.od_flag = 1
      AND t.cur = '810'
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0

      AND (
          t.TSEGMENTNAME <> N'ДЧБО'
          OR t.TSEGMENTNAME IS NULL
      )

      AND (
          (
              t.section_name = N'Накопительный счёт'
              AND t.acc_role = 'LIAB'
          )
          OR
          (
              t.section_name = N'Срочные'

              AND (
                  t.PROD_NAME_res IS NULL
                  OR t.PROD_NAME_res NOT IN (
                      N'ДОМа надёжно',
                      N'Всё в ДОМ',
                      N'Надёжный прайм',
                      N'НадёжныйVIP',
                      N'Надёжный премиум',
                      N'Надёжный промо',
                      N'Надёжный старт',
                      N'Надёжный Т2',
                      N'Надёжный Мегафон',
                      N'Надёжный процент',
                      N'Могучий',
                      N'Надёжный'
                  )
              )
          )
      )

    GROUP BY c.client_group

)

SELECT
    ISNULL(client_group, N'ИТОГО MASS') AS client_group,

    SUM(depo_before) / 1e9 AS depo_3009,
    SUM(depo_after) / 1e9 AS depo_0411,
    SUM(depo_after - depo_before) / 1e9 AS depo_delta,

    SUM(ns_before) / 1e9 AS ns_3009,
    SUM(ns_after) / 1e9 AS ns_0411,
    SUM(ns_after - ns_before) / 1e9 AS ns_delta,

    SUM(depo_before + ns_before) / 1e9 AS total_3009,
    SUM(depo_after + ns_after) / 1e9 AS total_0411,

    SUM(depo_after + ns_after
      - depo_before - ns_before) / 1e9 AS total_delta

FROM balances

GROUP BY GROUPING SETS (
    (client_group),
    ()
)

ORDER BY GROUPING(client_group), client_group

OPTION (MAXDOP 1, RECOMPILE);
