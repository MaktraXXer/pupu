WITH base AS (
    SELECT
        dt_rep,
        cli_id,
        fu_only_overall,
        prod_name_res,
        sum_out_rub
    FROM alm_test.dbo.fu_vintage_results_ext WITH (NOLOCK)
),
-- Группировка по dt_rep и fu_only_overall
grouped AS (
    SELECT
        dt_rep,
        fu_only_overall,

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

        SUM(sum_out_rub) AS sum_out_rub_total,

        COUNT(DISTINCT CASE WHEN prod_name_res IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id END) AS cnt_clients_fu,

        COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id END) AS cnt_clients_non_fu,

        COUNT(DISTINCT cli_id) AS cnt_clients_total
    FROM base
    GROUP BY dt_rep, fu_only_overall
),
-- Добавляем строку "итого" по всем клиентам в месяц
totals AS (
    SELECT
        dt_rep,
        2 AS fu_only_overall,
        SUM(sum_out_rub_fu)       AS sum_out_rub_fu,
        SUM(sum_out_rub_non_fu)   AS sum_out_rub_non_fu,
        SUM(sum_out_rub_total)    AS sum_out_rub_total,
        SUM(cnt_clients_fu)       AS cnt_clients_fu,
        SUM(cnt_clients_non_fu)   AS cnt_clients_non_fu,
        SUM(cnt_clients_total)    AS cnt_clients_total
    FROM grouped
    GROUP BY dt_rep
)
-- Объединяем основной расчёт и итого
SELECT *
FROM grouped
UNION ALL
SELECT *
FROM totals
ORDER BY dt_rep, fu_only_overall;
