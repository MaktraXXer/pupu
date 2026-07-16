WITH src AS (
    SELECT
        t.*,

        -- Удобное текстовое представление комбинации из 21 флажка
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

        -- Сколько флажков включено
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

    FROM ehd.attr_DepoFLConditions t WITH (NOLOCK)
),
ranked AS (
    SELECT
        src.*,

        COUNT(*) OVER (
            PARTITION BY flags_mask
        ) AS combination_frequency,

        ROW_NUMBER() OVER (
            PARTITION BY flags_mask
            ORDER BY
                DT_OPEN_FACT DESC,
                DT_UPDATE DESC,
                CON_ID DESC
        ) AS rn
    FROM src
)
SELECT
    flags_mask,
    flags_count,
    combination_frequency,

    CON_ID,
    CLI_ID,
    DT_OPEN_FACT,
    DT_CLOSE_PLAN,
    DT_CLOSE_FACT,
    DEPOSIT_ADD_CONDITIONS,
    PROMO_CODE,
    PROMO_GROUP,
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
    flags_count,
    combination_frequency DESC,
    flags_mask;
