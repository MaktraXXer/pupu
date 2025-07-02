/* ===== 0. базовая выборка ===== */
WITH base AS (
    SELECT
        dt_rep,
        cli_id,
        fu_only_overall,
        fu_had_deposit_before,
        prod_name_res,
        sum_out_rub
    FROM alm_test.dbo.fu_vintage_results_ext WITH (NOLOCK)
),

/* ===== 1. агрегат по полной комбинации (0/1) × (0/1) ===== */
grp_core AS (
    SELECT
        dt_rep,
        fu_only_overall,
        fu_had_deposit_before,

        /* объёмы */
        SUM(CASE WHEN prod_name_res IN (
              N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
              N'Надёжный промо', N'Надёжный старт',
              N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END)                           AS vol_fu,

        SUM(CASE WHEN prod_name_res NOT IN (
              N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
              N'Надёжный промо', N'Надёжный старт',
              N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END)                           AS vol_non_fu,

        SUM(sum_out_rub)                                         AS vol_total,

        /* клиенты */
        COUNT(DISTINCT CASE WHEN prod_name_res IN (
              N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
              N'Надёжный промо', N'Надёжный старт',
              N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id END)                                       AS cli_fu,

        COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
              N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
              N'Надёжный промо', N'Надёжный старт',
              N'Надёжный Т2', N'Надёжный Мегафон'
        ) THEN cli_id END)                                       AS cli_non_fu,

        COUNT(DISTINCT cli_id)                                   AS cli_total
    FROM base
    GROUP BY dt_rep, fu_only_overall, fu_had_deposit_before
),
/* ===== 2. тотал по fu_only_overall (ставим 2) ===== */
tot_fu_only AS (
    SELECT
        dt_rep,
        2 AS fu_only_overall,
        fu_had_deposit_before,

        SUM(CASE WHEN prod_name_res IN (
              N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
              N'Надёжный промо',N'Надёжный старт',
              N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS [vol_fu],

        SUM(CASE WHEN prod_name_res NOT IN (
              N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
              N'Надёжный промо',N'Надёжный старт',
              N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS [vol_non_fu],

        SUM(sum_out_rub) AS [vol_total],

        COUNT(DISTINCT CASE WHEN prod_name_res IN (
              N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
              N'Надёжный промо',N'Надёжный старт',
              N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END) AS [cli_fu],

        COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
              N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
              N'Надёжный промо',N'Надёжный старт',
              N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END) AS [cli_non_fu],

        COUNT(DISTINCT cli_id) AS [cli_total]

    FROM base
    GROUP BY dt_rep, fu_had_deposit_before
),
/* ===== 3. тотал по fu_had_deposit_before (ставим 2) ===== */
tot_had_before AS (
    SELECT
        dt_rep,
        fu_only_overall,
        2            AS fu_had_deposit_before,
        SUM(vol_fu)     AS vol_fu,
        SUM(vol_non_fu) AS vol_non_fu,
        SUM(vol_total)  AS vol_total,
        SUM(cli_fu)     AS cli_fu,
        SUM(cli_non_fu) AS cli_non_fu,
        SUM(cli_total)  AS cli_total
    FROM grp_core
    GROUP BY dt_rep, fu_only_overall
),

/* ===== 4. гран-тотал (2,2) ===== */
tot_tot AS (
    SELECT
        dt_rep,
        2 AS fu_only_overall,
        2 AS fu_had_deposit_before,
        SUM(vol_fu)     AS vol_fu,
        SUM(vol_non_fu) AS vol_non_fu,
        SUM(vol_total)  AS vol_total,
        SUM(cli_fu)     AS cli_fu,
        SUM(cli_non_fu) AS cli_non_fu,
        SUM(cli_total)  AS cli_total
    FROM grp_core
    GROUP BY dt_rep
)

/* ===== 5. объединяем всё ===== */
, union_all AS (
    SELECT * FROM grp_core
    UNION ALL
    SELECT * FROM tot_fu_only
    UNION ALL
    SELECT * FROM tot_had_before
    UNION ALL
    SELECT * FROM tot_tot
)

/* ===== 6. человекочитаемые категории + вывод ===== */
SELECT
    dt_rep,

    CASE fu_only_overall
        WHEN 0 THEN N'У клиента были продукты Банка'
        WHEN 1 THEN N'У клиента были вклады только на ФУ'
        WHEN 2 THEN N'Тотал по FU_ONLY'
    END   AS Категория_FU_ONLY,

    CASE fu_had_deposit_before
        WHEN 0 THEN N'У клиента не было продуктов Банка до открытия на ФУ'
        WHEN 1 THEN N'У клиента были продукты Банка до открытия на ФУ'
        WHEN 2 THEN N'Тотал по HAD_BEFORE'
    END   AS Категория_HAD_BEFORE,

    vol_fu      AS [Объем на ФУ, руб.],
    vol_non_fu  AS [Объем не на ФУ, руб.],
    vol_total   AS [Объем всего, руб.],

    cli_fu      AS [Уникальных клиентов на ФУ],
    cli_non_fu  AS [Уникальных клиентов не на ФУ],
    cli_total   AS [Уникальных клиентов всего]

FROM union_all
ORDER BY dt_rep,
         Категория_FU_ONLY,
         Категория_HAD_BEFORE;

этот запрос сработает?
