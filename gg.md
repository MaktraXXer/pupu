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


WITH deposits_opened_from_3004 AS (
    SELECT
          b.dt_rep
        , b.con_id
        , b.cli_id
        , b.acc_no
        , b.dt_open
        , b.dt_close_plan
        , b.termdays
        , b.rate_con
        , b.conv
        , b.out_rub
        , b.out_rub / 1000000.0 AS out_rub_mln
        , b.PROD_NAME_res
        , b.TSEGMENTNAME

        -- флаг размера договора
        , CASE 
            WHEN b.out_rub >= 1500000 THEN 1 
            ELSE 0 
          END AS dep_size_ge_1_5m_flag

        -- флаг попадания под промо-условия по ставке / conv / сумме
        , CASE 
            WHEN b.conv = 'AT_THE_END'  AND b.out_rub >= 1500000 AND b.rate_con = 0.1450 THEN 1
            WHEN b.conv = 'AT_THE_END'  AND b.out_rub <  1500000 AND b.rate_con = 0.1430 THEN 1
            WHEN b.conv <> 'AT_THE_END' AND b.out_rub <  1500000 AND b.rate_con = 0.1410 THEN 1
            WHEN b.conv <> 'AT_THE_END' AND b.out_rub >= 1500000 AND b.rate_con = 0.1430 THEN 1
            ELSE 0
          END AS promo_rate_flag

        -- расшифровка условия
        , CASE 
            WHEN b.conv = 'AT_THE_END'  AND b.out_rub >= 1500000 AND b.rate_con = 0.1450 
                THEN N'AT_THE_END; >=1.5 млн; ставка 14.50%'
            WHEN b.conv = 'AT_THE_END'  AND b.out_rub <  1500000 AND b.rate_con = 0.1430 
                THEN N'AT_THE_END; <1.5 млн; ставка 14.30%'
            WHEN b.conv <> 'AT_THE_END' AND b.out_rub <  1500000 AND b.rate_con = 0.1410 
                THEN N'НЕ AT_THE_END; <1.5 млн; ставка 14.10%'
            WHEN b.conv <> 'AT_THE_END' AND b.out_rub >= 1500000 AND b.rate_con = 0.1430 
                THEN N'НЕ AT_THE_END; >=1.5 млн; ставка 14.30%'
            ELSE N'Не попал под промо-условие'
          END AS promo_rate_group
    FROM #base b
    WHERE
        b.dt_rep = '2026-05-02'
        AND b.section_name = N'Срочные'
        AND b.dt_open >= '2026-04-30'
),

clients_with_deposits_to_close AS (
    SELECT
          b.cli_id
        , COUNT(DISTINCT b.con_id) AS close_dep_cnt
        , SUM(b.out_rub) AS close_dep_out_rub
        , SUM(b.out_rub) / 1000000.0 AS close_dep_out_rub_mln
    FROM #base b
    WHERE
        b.dt_rep = '2026-04-29'
        AND b.section_name = N'Срочные'
        AND b.dt_open <= '2026-04-29'
        AND b.dt_close_plan BETWEEN '2026-04-30' AND '2026-05-02'
    GROUP BY
        b.cli_id
),

ns_by_client AS (
    SELECT
          b.cli_id
        , SUM(CASE WHEN b.dt_rep = '2026-04-29' THEN b.out_rub ELSE 0 END) AS ns_2904
        , SUM(CASE WHEN b.dt_rep = '2026-05-02' THEN b.out_rub ELSE 0 END) AS ns_0205
    FROM #base b
    WHERE
        b.section_name = N'Накопительный счёт'
    GROUP BY
        b.cli_id
)

SELECT
      d.dt_rep
    , d.con_id
    , d.cli_id
    , d.acc_no
    , d.dt_open
    , d.dt_close_plan
    , d.termdays
    , d.rate_con
    , d.conv
    , d.out_rub
    , d.out_rub_mln
    , d.PROD_NAME_res
    , d.TSEGMENTNAME

    -- атрибуты самого открытого вклада
    , d.dep_size_ge_1_5m_flag
    , d.promo_rate_flag
    , d.promo_rate_group

    -- атрибуты клиента по вкладам к выходу, размазанные на договор
    , CASE 
        WHEN c.cli_id IS NOT NULL THEN 1 
        ELSE 0 
      END AS client_had_dep_to_close_flag
    , ISNULL(c.close_dep_cnt, 0) AS client_close_dep_cnt
    , ISNULL(c.close_dep_out_rub, 0) AS client_close_dep_out_rub
    , ISNULL(c.close_dep_out_rub_mln, 0) AS client_close_dep_out_rub_mln

    -- атрибуты клиента по НС, размазанные на договор
    , ISNULL(ns.ns_2904, 0) AS client_ns_2904
    , ISNULL(ns.ns_0205, 0) AS client_ns_0205
    , ISNULL(ns.ns_0205, 0) - ISNULL(ns.ns_2904, 0) AS client_ns_delta

    , CASE 
        WHEN ns.cli_id IS NULL THEN 1 
        ELSE 0 
      END AS client_had_no_ns_flag

    , CASE 
        WHEN ISNULL(ns.ns_0205, 0) < ISNULL(ns.ns_2904, 0) THEN 1 
        ELSE 0 
      END AS client_ns_decrease_flag

    , CASE 
        WHEN ISNULL(ns.ns_0205, 0) > ISNULL(ns.ns_2904, 0) THEN 1 
        ELSE 0 
      END AS client_ns_increase_flag

    , CASE 
        WHEN ns.cli_id IS NOT NULL 
             AND ISNULL(ns.ns_0205, 0) = ISNULL(ns.ns_2904, 0) THEN 1 
        ELSE 0 
      END AS client_ns_unchanged_flag

    -- итоговый флаг чистого клиента по нашей логике
    , CASE 
        WHEN c.cli_id IS NULL
         AND NOT (ISNULL(ns.ns_0205, 0) < ISNULL(ns.ns_2904, 0))
            THEN 1
        ELSE 0
      END AS clean_client_flag

FROM deposits_opened_from_3004 d
LEFT JOIN clients_with_deposits_to_close c
    ON c.cli_id = d.cli_id
LEFT JOIN ns_by_client ns
    ON ns.cli_id = d.cli_id
ORDER BY
      d.promo_rate_flag DESC
    , clean_client_flag DESC
    , d.out_rub DESC;
