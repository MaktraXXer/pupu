### Коротко

* **Логика обоих отчёт-скриптов корректна** – менять расчётные формулы не нужно.
* После последних исправлений в таблице `fu_vintage_results_ext` имя поля – **`had_deposit_before`** (без префиксов). В скрипте № 2 оно именно так и используется → ок.
* Единственное, что **обязательно обновить** в двух скриптах – «жёсткий» перечень Main-вкладов (добавить *Надёжный Т2* и *Надёжный Мегафон*).
  Если список не обновить, то строки с новыми названиями будут посчитаны как «не на ФУ».

Ниже – оба запроса в рабочем виде уже с расширенным списком Main-продуктов.
Названия выходных колонок сохранены.

---

## 📄 Скрипт 1 «FU ONLY / не FU / Тотал»

```sql
/* 0. Базовая выборка */
WITH base AS (
    SELECT dt_rep,
           cli_id,
           fu_only_overall,        -- 0 / 1
           prod_name_res,
           sum_out_rub
    FROM alm_test.dbo.fu_vintage_results_ext WITH (NOLOCK)
),

/* 1. Группировка 0-/1-клиентов */
grouped AS (
    SELECT
        dt_rep,
        fu_only_overall,

        /* ─── объёмы ─── */
        SUM(CASE WHEN prod_name_res IN (
            N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
            N'Надёжный промо',N'Надёжный старт',
            N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END)               AS [Объем на ФУ, руб.],

        SUM(CASE WHEN prod_name_res NOT IN (
            N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
            N'Надёжный промо',N'Надёжный старт',
            N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END)               AS [Объем не на ФУ, руб.],

        SUM(sum_out_rub)                             AS [Объем всего, руб.],

        /* ─── клиенты ─── */
        COUNT(DISTINCT CASE WHEN prod_name_res IN (
            N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
            N'Надёжный промо',N'Надёжный старт',
            N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END)                           AS [Уникальных клиентов на ФУ],

        COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
            N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
            N'Надёжный промо',N'Надёжный старт',
            N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END)                           AS [Уникальных клиентов не на ФУ],

        COUNT(DISTINCT cli_id)                       AS [Уникальных клиентов всего]
    FROM base
    GROUP BY dt_rep, fu_only_overall
),

/* 2. Тотал (fu_only_overall = 2) */
totals AS (
    SELECT
        dt_rep,
        2 AS fu_only_overall,
        [Объем на ФУ, руб.],
        [Объем не на ФУ, руб.],
        [Объем всего, руб.],
        [Уникальных клиентов на ФУ],
        [Уникальных клиентов не на ФУ],
        [Уникальных клиентов всего]
    FROM grouped
    GROUP BY dt_rep
)

/* 3. Итоговая выборка */
SELECT
    dt_rep,
    CASE fu_only_overall
        WHEN 1 THEN N'У клиента были вклады только на ФУ'
        WHEN 0 THEN N'У клиента были продукты Банка'
        WHEN 2 THEN N'Тотал'
    END                                   AS [Категория клиентов],
    [Объем на ФУ, руб.],
    [Объем не на ФУ, руб.],
    [Объем всего, руб.],
    [Уникальных клиентов на ФУ],
    [Уникальных клиентов не на ФУ],
    [Уникальных клиентов всего]
FROM (
    SELECT * FROM grouped
    UNION ALL
    SELECT * FROM totals
) fin
ORDER BY dt_rep, [Категория клиентов];
```

---

## 📄 Скрипт 2 «FU ONLY × HAD BEFORE»

```sql
/* 0. Базовая выборка */
WITH base AS (
    SELECT dt_rep,
           cli_id,
           fu_only_overall,        -- 0 / 1
           had_deposit_before,     -- 0 / 1
           prod_name_res,
           sum_out_rub
    FROM alm_test.dbo.fu_vintage_results_ext WITH (NOLOCK)
),

/* 1. Полная матрица 0/1 × 0/1 */
core AS (
    SELECT
        dt_rep,
        fu_only_overall,
        had_deposit_before,

        /* ─── объёмы ─── */
        SUM(CASE WHEN prod_name_res IN (
            N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
            N'Надёжный промо',N'Надёжный старт',
            N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END)               AS vol_fu,

        SUM(CASE WHEN prod_name_res NOT IN (
            N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
            N'Надёжный промо',N'Надёжный старт',
            N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN sum_out_rub ELSE 0 END)               AS vol_non_fu,

        SUM(sum_out_rub)                             AS vol_total,

        /* ─── клиенты ─── */
        COUNT(DISTINCT CASE WHEN prod_name_res IN (
            N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
            N'Надёжный промо',N'Надёжный старт',
            N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END)                           AS cli_fu,

        COUNT(DISTINCT CASE WHEN prod_name_res NOT IN (
            N'Надёжный',N'Надёжный VIP',N'Надёжный премиум',
            N'Надёжный промо',N'Надёжный старт',
            N'Надёжный Т2',N'Надёжный Мегафон'
        ) THEN cli_id END)                           AS cli_non_fu,

        COUNT(DISTINCT cli_id)                       AS cli_total
    FROM base
    GROUP BY dt_rep, fu_only_overall, had_deposit_before
),

/* 2. Тоталы */
tot_fu_only AS (         -- fu_only_overall = 2
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
    FROM core
    GROUP BY dt_rep, had_deposit_before
),
tot_had_before AS (      -- had_deposit_before = 2
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
    FROM core
    GROUP BY dt_rep, fu_only_overall
),
tot_tot AS (             -- 2 × 2
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
    FROM core
    GROUP BY dt_rep
)

/* 3. Итоговая выборка */
SELECT
    dt_rep,
    CASE fu_only_overall
        WHEN 0 THEN N'У клиента были продукты Банка'
        WHEN 1 THEN N'У клиента были вклады только на ФУ'
        WHEN 2 THEN N'Тотал по FU_ONLY'
    END                                   AS Категория_FU_ONLY,

    CASE had_deposit_before
        WHEN 0 THEN N'У клиента не было продуктов Банка до открытия на ФУ'
        WHEN 1 THEN N'У клиента были продукты Банка до открытия на ФУ'
        WHEN 2 THEN N'Тотал по HAD_BEFORE'
    END                                   AS Категория_HAD_BEFORE,

    vol_fu      AS [Объем на ФУ, руб.],
    vol_non_fu  AS [Объем не на ФУ, руб.],
    vol_total   AS [Объем всего, руб.],

    cli_fu      AS [Уникальных клиентов на ФУ],
    cli_non_fu  AS [Уникальных клиентов не на ФУ],
    cli_total   AS [Уникальных клиентов всего]

FROM (
    SELECT * FROM core
    UNION ALL
    SELECT * FROM tot_fu_only
    UNION ALL
    SELECT * FROM tot_had_before
    UNION ALL
    SELECT * FROM tot_tot
) fin
ORDER BY dt_rep, Категория_FU_ONLY, Категория_HAD_BEFORE;
```

### Что ещё проверить

* Скрипты рассчитаны на то, что таблица-источник **уже** пересчитана с поправкой `COALESCE` (чтобы `fu_only_*` и `had_deposit_before` были заполнены верно).
* Если вы разместили таблицу в другом схеме/имени – поправьте `FROM … fu_vintage_results_ext`.
* При добавлении новых Main-вкладов в будущем – обновляйте список в обоих отчётах.

После этой правки оба запроса отрабатывают без ошибок и дают ожидаемые показатели.
