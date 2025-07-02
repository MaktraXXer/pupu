### Нужно ли менять скрипты?

* **`fu_only_overall`** — остался тем же (1 = «только Main-вклады», 0 = появился хотя бы один вклад вне Main).
  → оба запроса уже используют именно это понимание, менять логику **не нужно**.
* **`had_deposit_before`** теперь означает «был *любой* вклад раньше первого Main».
  В скрипте № 2 он используется ровно так, поэтому логику **трогать не нужно**.
* Единственное, что стоит обновить — «жёсткий» список Main-вкладов в обоих
  запросах: добавить `Надёжный T2` и `Надёжный Мегафон`, чтобы счётчики
  «на ФУ / не на ФУ» точно совпадали с тем, как вы формируете табличку.

Ниже оба скрипта уже с расширенным списком Main-продуктов.
Названия выходных полей сохранены.

---

## Скрипт 1 — «FU ONLY / не FU / тотал»

```sql
/* 0. Базовый выбор */
WITH base AS (
    SELECT dt_rep, cli_id,
           fu_only_overall,
           prod_name_res,
           sum_out_rub
    FROM alm_test.dbo.fu_vintage_results_ext WITH (NOLOCK)
),

/* 1. Группировка 0 / 1 */
grouped AS (
    SELECT
        dt_rep,
        fu_only_overall,

        /* объёмы */
        SUM(CASE WHEN prod_name_res IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS [Объем на ФУ, руб.],

        SUM(CASE WHEN prod_name_res NOT IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS [Объем не на ФУ, руб.],

        SUM(sum_out_rub) AS [Объем всего, руб.],

        /* клиенты */
        COUNT(DISTINCT CASE WHEN prod_name_res IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END) AS [Уникальных клиентов на ФУ],

        COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END) AS [Уникальных клиентов не на ФУ],

        COUNT(DISTINCT cli_id) AS [Уникальных клиентов всего]
    FROM base
    GROUP BY dt_rep, fu_only_overall
),

/* 2. Тотал по FU_ONLY (ставим 2) */
totals AS (
    SELECT
        dt_rep,
        2 AS fu_only_overall,

        /* объёмы / клиенты */
        SUM(CASE WHEN prod_name_res IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS [Объем на ФУ, руб.],

        SUM(CASE WHEN prod_name_res NOT IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS [Объем не на ФУ, руб.],

        SUM(sum_out_rub) AS [Объем всего, руб.],

        COUNT(DISTINCT CASE WHEN prod_name_res IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END) AS [Уникальных клиентов на ФУ],

        COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END) AS [Уникальных клиентов не на ФУ],

        COUNT(DISTINCT cli_id) AS [Уникальных клиентов всего]
    FROM base
    GROUP BY dt_rep
),

/* 3. Склеиваем */
final AS (
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
FROM final
ORDER BY dt_rep, [Категория клиентов];
```

---

## Скрипт 2 — «FU\_ONLY × HAD\_BEFORE»

```sql
/* 0. Базовая выборка */
WITH base AS (
    SELECT dt_rep, cli_id,
           fu_only_overall,
           had_deposit_before,
           prod_name_res,
           sum_out_rub
    FROM alm_test.dbo.fu_vintage_results_ext WITH (NOLOCK)
),

/* 1. Полная матрица 0/1 × 0/1 */
grp_core AS (
    SELECT
        dt_rep,
        fu_only_overall,
        had_deposit_before,

        /* объёмы */
        SUM(CASE WHEN prod_name_res IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS vol_fu,

        SUM(CASE WHEN prod_name_res NOT IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END) AS vol_non_fu,

        SUM(sum_out_rub) AS vol_total,

        /* клиенты */
        COUNT(DISTINCT CASE WHEN prod_name_res IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END) AS cli_fu,

        COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
             N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
             N'Надёжный промо',N'Надёжный старт',
             N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END) AS cli_non_fu,

        COUNT(DISTINCT cli_id) AS cli_total
    FROM base
    GROUP BY dt_rep, fu_only_overall, had_deposit_before
),

/* 2. Тотал по FU_ONLY */
tot_fu_only AS (
    SELECT
        dt_rep,
        2 AS fu_only_overall,
        had_deposit_before,
        SUM(vol_fu)     AS vol_fu,
        SUM(vol_non_fu) AS vol_non_fu,
        SUM(vol_total)  AS vol_total,
        SUM(cli_fu)     AS cli_fu,
        SUM(cli_non_fu) AS cli_non_fu,
        SUM(cli_total)  AS cli_total
    FROM grp_core
    GROUP BY dt_rep, had_deposit_before
),

/* 3. Тотал по HAD_BEFORE */
tot_had_before AS (
    SELECT
        dt_rep,
        fu_only_overall,
        2 AS had_deposit_before,
        SUM(vol_fu)     AS vol_fu,
        SUM(vol_non_fu) AS vol_non_fu,
        SUM(vol_total)  AS vol_total,
        SUM(cli_fu)     AS cli_fu,
        SUM(cli_non_fu) AS cli_non_fu,
        SUM(cli_total)  AS cli_total
    FROM grp_core
    GROUP BY dt_rep, fu_only_overall
),

/* 4. Гран-тотал 2×2 (объёмы + уник. клиенты) */
tot_tot AS (
    SELECT
        dt_rep,
        2 AS fu_only_overall,
        2 AS had_deposit_before,
        SUM(vol_fu)     AS vol_fu,
        SUM(vol_non_fu) AS vol_non_fu,
        SUM(vol_total)  AS vol_total,
        SUM(cli_fu)     AS cli_fu,
        SUM(cli_non_fu) AS cli_non_fu,
        SUM(cli_total)  AS cli_total
    FROM grp_core
    GROUP BY dt_rep
),

/* 5. Объединяем */
union_all AS (
    SELECT * FROM grp_core
    UNION ALL
    SELECT * FROM tot_fu_only
    UNION ALL
    SELECT * FROM tot_had_before
    UNION ALL
    SELECT * FROM tot_tot
)

/* 6. Человекочитаемые категории */
SELECT
    dt_rep,

    CASE fu_only_overall
        WHEN 0 THEN N'У клиента были продукты Банка'
        WHEN 1 THEN N'У клиента были вклады только на ФУ'
        WHEN 2 THEN N'Тотал по FU_ONLY'
    END AS Категория_FU_ONLY,

    CASE had_deposit_before
        WHEN 0 THEN N'У клиента не было продуктов Банка до открытия на ФУ'
        WHEN 1 THEN N'У клиента были продукты Банка до открытия на ФУ'
        WHEN 2 THEN N'Тотал по HAD_BEFORE'
    END AS Категория_HAD_BEFORE,

    vol_fu      AS [Объем на ФУ, руб.],
    vol_non_fu  AS [Объем не на ФУ, руб.],
    vol_total   AS [Объем всего, руб.],

    cli_fu      AS [Уникальных клиентов на ФУ],
    cli_non_fu  AS [Уникальных клиентов не на ФУ],
    cli_total   AS [Уникальных клиентов всего]

FROM union_all
ORDER BY dt_rep, Категория_FU_ONLY, Категория_HAD_BEFORE;
```

Оба запроса уже используют флаги в нужном смысле; изменилась лишь «жёсткая» константа со списком Main-продуктов — она дополнена новыми названиями.
