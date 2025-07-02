WITH base AS (
    SELECT
        dt_rep,
        fu_only_overall,
        prod_name_res,
        cli_id,
        sum_out_rub
    FROM alm_test.dbo.fu_vintage_results_ext WITH (NOLOCK)
),
unioned AS (
    SELECT * FROM base
    UNION ALL
    SELECT
        dt_rep,
        2 AS fu_only_overall,  -- общий итог
        prod_name_res,
        cli_id,
        sum_out_rub
    FROM base
)
SELECT
    dt_rep,
    fu_only_overall,

    -- Объемы
    SUM(CASE WHEN prod_name_res IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS sum_out_rub_fu,

    SUM(CASE WHEN prod_name_res NOT IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS sum_out_rub_non_fu,

    -- Клиенты
    COUNT(DISTINCT CASE WHEN prod_name_res IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id END) AS cnt_clients_fu,

    COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id END) AS cnt_clients_non_fu

FROM unioned
GROUP BY
    dt_rep,
    fu_only_overall
ORDER BY
    dt_rep,
    fu_only_overall;
