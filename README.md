WITH src AS (
    SELECT
        t.*,

        CONCAT(
            CASE WHEN [Зрп] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Пнс] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Нов] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [НДП] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [НДМ] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Мпл] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Прл] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Пр2] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Пр3] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Пк2] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [От1] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [От2] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [От3] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Пк3] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [ДБО] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Лмт] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Прм] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Пк1] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Пк4] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Пк5] = 1 THEN '1' ELSE '0' END,
            CASE WHEN [Пк6] = 1 THEN '1' ELSE '0' END
        ) AS flags_mask,

        CASE WHEN [Зрп] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Пнс] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Нов] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [НДП] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [НДМ] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Мпл] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Прл] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Пр2] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Пр3] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Пк2] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [От1] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [От2] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [От3] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Пк3] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [ДБО] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Лмт] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Прм] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Пк1] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Пк4] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Пк5] = 1 THEN 1 ELSE 0 END +
        CASE WHEN [Пк6] = 1 THEN 1 ELSE 0 END AS flags_count

    FROM ehd.attr_DepoFLConditions AS t WITH (NOLOCK)
),
ranked AS (
    SELECT
        src.*,

        COUNT(*) OVER (
            PARTITION BY PROMO_GROUP, flags_mask
        ) AS combination_frequency,

        MIN(DT_OPEN_FACT) OVER (
            PARTITION BY PROMO_GROUP, flags_mask
        ) AS first_dt_open_fact,

        MAX(DT_OPEN_FACT) OVER (
            PARTITION BY PROMO_GROUP, flags_mask
        ) AS last_dt_open_fact,

        SUM(COALESCE(START_DEPOSIT, 0)) OVER (
            PARTITION BY PROMO_GROUP, flags_mask
        ) AS total_start_deposit,

        SUM(
            CASE
                WHEN DT_OPEN_FACT >= DATEFROMPARTS(2026, 1, 1)
                THEN 1
                ELSE 0
            END
        ) OVER (
            PARTITION BY PROMO_GROUP, flags_mask
        ) AS deposits_count_from_2026,

        SUM(
            CASE
                WHEN DT_OPEN_FACT >= DATEFROMPARTS(2026, 1, 1)
                THEN COALESCE(START_DEPOSIT, 0)
                ELSE 0
            END
        ) OVER (
            PARTITION BY PROMO_GROUP, flags_mask
        ) AS start_deposit_from_2026,

        ROW_NUMBER() OVER (
            PARTITION BY PROMO_GROUP, flags_mask
            ORDER BY
                DT_OPEN_FACT DESC,
                START_DEPOSIT DESC,
                DT_UPDATE DESC,
                CON_ID DESC
        ) AS rn

    FROM src
)
SELECT
    PROMO_GROUP,

    flags_mask,
    flags_count,
    combination_frequency,

    first_dt_open_fact,
    last_dt_open_fact,

    total_start_deposit,
    deposits_count_from_2026,
    start_deposit_from_2026,

    -- Пример самого свежего и наиболее крупного вклада
    CON_ID,
    CLI_ID,
    DT_OPEN_FACT,
    DT_CLOSE_PLAN,
    DT_CLOSE_FACT,
    DEPOSIT_ADD_CONDITIONS,
    PROMO_CODE,
    START_DEPOSIT,
    selected_options,

    [Зрп],
    [Пнс],
    [Нов],
    [НДП],
    [НДМ],
    [Мпл],
    [Прл],
    [Пр2],
    [Пр3],
    [Пк2],
    [От1],
    [От2],
    [От3],
    [Пк3],
    [ДБО],
    [Лмт],
    [Прм],
    [Пк1],
    [Пк4],
    [Пк5],
    [Пк6],

    DT_UPDATE,
    loaddate
FROM ranked
WHERE rn = 1
ORDER BY
    PROMO_GROUP,
    flags_count,
    combination_frequency DESC,
    flags_mask;
