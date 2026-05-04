WITH a AS (
    SELECT
          t.dt_rep
        , t.con_id
        , t.acc_no
        , t.termdays
        , t.rate_con
        , t.conv
        , CASE 
            WHEN t.conv = 'AT_THE_END'  AND t.out_rub >= 1500000 AND t.rate_con = 0.1450 THEN 1
            WHEN t.conv = 'AT_THE_END'  AND t.out_rub <  1500000 AND t.rate_con = 0.1430 THEN 1
            WHEN t.conv <> 'AT_THE_END' AND t.out_rub <  1500000 AND t.rate_con = 0.1410 THEN 1
            WHEN t.conv <> 'AT_THE_END' AND t.out_rub >= 1500000 AND t.rate_con = 0.1430 THEN 1
            ELSE 0
          END AS promo_flag
        , t.cli_id
        , t.dt_open
        , t.out_rub / 1000000.0 AS out_rub_mln
        , CASE WHEN t.out_rub >= 1500000 THEN 1 ELSE 0 END AS [РазмерВклада-флаг]
        , t.PROD_NAME_res
        , t.TSEGMENTNAME
    FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
    WHERE
        t.dt_rep = '2026-05-02'
        AND t.dt_open >= '2026-04-30'
        AND t.section_name = N'Срочные'
        AND t.block_name   = N'Привлечение ФЛ'
        AND t.od_flag      = 1
        AND t.cur          = '810'
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
        AND t.termdays BETWEEN 60 AND 78
),

clients_with_deposits_to_close AS (
    SELECT DISTINCT
        t.con_id
    FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
    WHERE
        t.dt_rep = '2026-04-29'
        AND t.section_name = N'Срочные'
        AND t.block_name   = N'Привлечение ФЛ'
        AND t.od_flag      = 1
        AND t.cur          = '810'
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
        AND t.dt_open <= '2026-04-29'
        AND t.dt_close_plan BETWEEN '2026-04-30' AND '2026-05-02'
),

ns_by_client AS (
    SELECT
          t.cli_id
        , SUM(CASE WHEN t.dt_rep = '2026-04-29' THEN t.out_rub ELSE 0 END) AS ns_2904
        , SUM(CASE WHEN t.dt_rep = '2026-05-02' THEN t.out_rub ELSE 0 END) AS ns_0205
    FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
    WHERE
        t.dt_rep IN ('2026-04-29', '2026-05-02')
        AND t.section_name = N'Накопительный счёт'
        AND t.block_name   = N'Привлечение ФЛ'
        AND t.od_flag      = 1
        AND t.cur          = '810'
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
    GROUP BY
        t.cli_id
),

clients_with_ns_decrease AS (
    SELECT
        cli_id
    FROM ns_by_client
    WHERE ns_0205 < ns_2904
)

SELECT
    SUM(a.out_rub_mln) AS total_out_rub_mln
FROM a
WHERE
    a.promo_flag = 1
    AND a.con_id NOT IN (
        SELECT con_id
        FROM clients_with_deposits_to_close
    )
    AND a.cli_id NOT IN (
        SELECT cli_id
        FROM clients_with_ns_decrease
    );
