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

    -- Клиенты: считаем DISTINCT только если у клиента был хотя бы один вклад нужного типа
    COUNT(DISTINCT CASE WHEN prod_name_res IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id ELSE NULL END) AS cnt_clients_fu,

    COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id ELSE NULL END) AS cnt_clients_non_fu

FROM alm_test.dbo.fu_vintage_results_ext WITH (NOLOCK)

GROUP BY
    dt_rep,
    fu_only_overall

ORDER BY
    dt_rep,
    fu_only_overall;
