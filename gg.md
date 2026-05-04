IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;

SELECT
      t.dt_rep
    , t.con_id
    , t.cli_id
    , t.acc_no
    , t.termdays
    , t.rate_con
    , t.conv
    , t.dt_open
    , t.dt_close_plan
    , t.section_name
    , t.out_rub
    , t.PROD_NAME_res
    , t.TSEGMENTNAME
INTO #base
FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
WHERE
    t.dt_rep IN ('2026-04-29', '2026-05-02')
    AND t.block_name = N'Привлечение ФЛ'
    AND t.od_flag = 1
    AND t.cur = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
    AND t.section_name IN (N'Срочные', N'Накопительный счёт');


WITH new_promo_deposits AS (
    SELECT
          b.con_id
        , b.cli_id
        , b.out_rub / 1000000.0 AS out_rub_mln
    FROM #base b
    WHERE
        b.dt_rep = '2026-05-02'
        AND b.section_name = N'Срочные'
        AND b.dt_open >= '2026-04-30'
        AND b.termdays BETWEEN 60 AND 78
        AND (
               b.conv = 'AT_THE_END'  AND b.out_rub >= 1500000 AND b.rate_con = 0.1450
            OR b.conv = 'AT_THE_END'  AND b.out_rub <  1500000 AND b.rate_con = 0.1430
            OR b.conv <> 'AT_THE_END' AND b.out_rub <  1500000 AND b.rate_con = 0.1410
            OR b.conv <> 'AT_THE_END' AND b.out_rub >= 1500000 AND b.rate_con = 0.1430
        )
),

clients_with_deposits_to_close AS (
    SELECT DISTINCT
        b.cli_id
    FROM #base b
    WHERE
        b.dt_rep = '2026-04-29'
        AND b.section_name = N'Срочные'
        AND b.dt_open <= '2026-04-29'
        AND b.dt_close_plan BETWEEN '2026-04-30' AND '2026-05-02'
),

clients_with_ns_decrease AS (
    SELECT
        b.cli_id
    FROM #base b
    WHERE
        b.section_name = N'Накопительный счёт'
    GROUP BY
        b.cli_id
    HAVING
        SUM(CASE WHEN b.dt_rep = '2026-05-02' THEN b.out_rub ELSE 0 END)
        <
        SUM(CASE WHEN b.dt_rep = '2026-04-29' THEN b.out_rub ELSE 0 END)
)

SELECT
    SUM(n.out_rub_mln) AS total_out_rub_mln
FROM new_promo_deposits n
WHERE
    NOT EXISTS (
        SELECT 1
        FROM clients_with_deposits_to_close c
        WHERE c.cli_id = n.cli_id
    )
    AND NOT EXISTS (
        SELECT 1
        FROM clients_with_ns_decrease ns
        WHERE ns.cli_id = n.cli_id
    );
