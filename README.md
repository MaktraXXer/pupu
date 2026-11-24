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
