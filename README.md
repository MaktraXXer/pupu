WITH base AS (
    SELECT
        dt_rep,
        cli_id,
        fu_only_overall,
        prod_name_res,
        sum_out_rub
    FROM alm_test.dbo.fu_vintage_results_ext WITH (NOLOCK)
),
grouped AS (
    SELECT
        dt_rep,
        fu_only_overall,

        SUM(CASE WHEN prod_name_res IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS [Объем на ФУ, руб.],

        SUM(CASE WHEN prod_name_res NOT IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS [Объем не на ФУ, руб.],

        SUM(sum_out_rub) AS [Объем всего, руб.],

        COUNT(DISTINCT CASE WHEN prod_name_res IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id END) AS [Уникальных клиентов на ФУ],

        COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id END) AS [Уникальных клиентов не на ФУ],

        COUNT(DISTINCT cli_id) AS [Уникальных клиентов всего]
    FROM base
    GROUP BY dt_rep, fu_only_overall
),
totals AS (
    SELECT
        dt_rep,
        2 AS fu_only_overall,
        SUM([Объем на ФУ, руб.])            AS [Объем на ФУ, руб.],
        SUM([Объем не на ФУ, руб.])         AS [Объем не на ФУ, руб.],
        SUM([Объем всего, руб.])            AS [Объем всего, руб.],
        SUM([Уникальных клиентов на ФУ])    AS [Уникальных клиентов на ФУ],
        SUM([Уникальных клиентов не на ФУ]) AS [Уникальных клиентов не на ФУ],
        SUM([Уникальных клиентов всего])    AS [Уникальных клиентов всего]
    FROM grouped
    GROUP BY dt_rep
),
all_data AS (
    SELECT * FROM grouped
    UNION ALL
    SELECT * FROM totals
)
SELECT
    dt_rep,
    CASE fu_only_overall
        WHEN 1 THEN N'У клиента были вклады только на ФУ'
        WHEN 0 THEN N'У клиента были продукты Банка'
        WHEN 2 THEN N'Тотал'
    END AS [Категория клиентов],
    [Объем на ФУ, руб.],
    [Объем не на ФУ, руб.],
    [Объем всего, руб.],
    [Уникальных клиентов на ФУ],
    [Уникальных клиентов не на ФУ],
    [Уникальных клиентов всего]
FROM all_data
ORDER BY dt_rep, [Категория клиентов];

