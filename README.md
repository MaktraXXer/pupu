SELECT
    dt_rep,
    CASE 
        WHEN fu_only_overall IN (0, 1) THEN CAST(fu_only_overall AS varchar)
        ELSE '2'  -- Тотал строка
    END AS fu_only_overall_group,

    -- Объемы: только ФУ-вклады
    SUM(CASE WHEN prod_name_res IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS sum_out_rub_fu,

    -- Объемы: не-ФУ
    SUM(CASE WHEN prod_name_res NOT IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS sum_out_rub_non_fu,

    -- Объемы: всего
    SUM(sum_out_rub) AS sum_out_rub_total,

    -- Уникальные клиенты на ФУ
    COUNT(DISTINCT CASE WHEN prod_name_res IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id ELSE NULL END) AS cnt_clients_fu,

    -- Уникальные клиенты на не-ФУ
    COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id ELSE NULL END) AS cnt_clients_non_fu,

    -- Уникальные клиенты всего
    COUNT(DISTINCT cli_id) AS cnt_clients_total

FROM (
    SELECT *
    FROM alm_test.dbo.fu_vintage_results_ext WITH (NOLOCK)
    UNION ALL
    SELECT
        dt_rep,
        cli_id,
        2 AS fu_only_overall,  -- Обозначение тотал-строки
        NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL,
        prod_name_res,
        section_name,
        tsegmentname,
        sum_out_rub,
        count_con_id,
        rate_obiem,
        ts_obiem,
        avg_rate_con,
        avg_rate_trf,
        load_timestamp
    FROM alm_test.dbo.fu_vintage_results_ext WITH (NOLOCK)
) AS x
GROUP BY
    dt_rep,
    CASE 
        WHEN fu_only_overall IN (0, 1) THEN fu_only_overall
        ELSE 2
    END
ORDER BY dt_rep, fu_only_overall_group;
