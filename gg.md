DECLARE @DT_FROM date = '2025-01-01';
DECLARE @DT_TO   date = DATEADD(day, -2, CAST(GETDATE() AS date));

WITH calendar AS (
    SELECT 
        CAST([Date] AS date) AS [Date]
    FROM ALM.info.VW_calendar WITH (NOLOCK)
    WHERE [Date] BETWEEN @DT_FROM AND @DT_TO
),

together AS (
    SELECT
        'НС' AS section_name,
        CAST(dt_rep AS date) AS dt_rep,
        data_scope,
        out_rub_total,
        rate_con,
        CAST(0 AS int) AS has_matur,
        CAST(NULL AS decimal(18, 6)) AS matur
    FROM ALM_TEST.mail.balance_metrics_savings WITH (NOLOCK)

    UNION ALL

    SELECT
        'ДВС' AS section_name,
        CAST(dt_rep AS date) AS dt_rep,
        data_scope,
        out_rub_total,
        CASE 
            WHEN data_scope = 'новые'
                THEN FIRST_VALUE(rate_con) OVER (
                    PARTITION BY dt_rep
                    ORDER BY CASE WHEN data_scope = 'портфель' THEN 0 ELSE 1 END
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
                )
            ELSE rate_con
        END AS rate_con,
        CAST(0 AS int) AS has_matur,
        CAST(NULL AS decimal(18, 6)) AS matur
    FROM ALM_TEST.mail.balance_metrics_dvs WITH (NOLOCK)
),

DVS_FL AS (
    SELECT
        t.section_name,
        t.dt_rep,
        t.data_scope,
        t.out_rub_total,
        t.rate_con + 0.0048 AS rate_con,
        kr.KEY_RATE,
        t.rate_con + 0.0048 - kr.KEY_RATE AS spread_keyrate,
        t.has_matur,
        t.matur
    FROM together t
    LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache kr WITH (NOLOCK)
        ON t.dt_rep = kr.dt_rep
       AND kr.term = 1
    WHERE t.dt_rep >= @DT_FROM
),

v_fl_fix AS (
    /* портфель фиксированных вкладов ФЛ:
       оставляем старую готовую строку all termgroup,
       matur не выводим */
    SELECT
        'вклады ФЛ с фикс ставкой' AS section_name,
        CAST(t.[Date] AS date) AS dt_rep,
        'портфель' AS data_scope,
        t.balance_rub AS out_rub_total,
        t.[MonthlyCONV_RATE_SSV] AS rate_con,
        t.[MonthlyCONV_ForecastKeyRate] AS KEY_RATE,
        t.[Spread_KeyRate] AS spread_keyrate,
        CAST(1 AS int) AS has_matur,
        CAST(NULL AS decimal(18, 6)) AS matur
    FROM [ALM_TEST].[WORK].[vGroupDepositInterestsRate_For_FL] t WITH (NOLOCK)
    WHERE t.[Date] >= @DT_FROM
      AND t.[Date] <= @DT_TO
      AND t.[Признак опциональности для ФЛ] = 'all option_type'
      AND t.[Сегмент ФЛ] = 'all segment'
      AND t.[Срочн. бакеты] = 'all termgroup'
      AND t.[Тип маржи] = 'all margin'
      AND t.[Сегмент бизнеса] = 'all segment'
      AND t.[Тип клиента] = 'all client'
      AND t.[Тип отчета] = 'Срез'
      AND t.[Спред к КС бакеты] = 'all spread bucket'

    UNION ALL

    /* новые фиксированные вклады ФЛ:
       берем строки по срочным бакетам,
       rate_con / KEY_RATE / spread_keyrate считаем с весом balance_rub * MATUR,
       matur считаем с весом balance_rub */
    SELECT
        'вклады ФЛ с фикс ставкой' AS section_name,
        CAST(t.[Date] AS date) AS dt_rep,
        'новые' AS data_scope,

        SUM(t.balance_rub) AS out_rub_total,

        SUM(t.[MonthlyCONV_RATE_SSV] * t.balance_rub * t.[MATUR])
            / NULLIF(SUM(t.balance_rub * t.[MATUR]), 0) AS rate_con,

        SUM(t.[MonthlyCONV_ForecastKeyRate] * t.balance_rub * t.[MATUR])
            / NULLIF(SUM(t.balance_rub * t.[MATUR]), 0) AS KEY_RATE,

        SUM(t.[Spread_KeyRate] * t.balance_rub * t.[MATUR])
            / NULLIF(SUM(t.balance_rub * t.[MATUR]), 0) AS spread_keyrate,

        CAST(1 AS int) AS has_matur,

        SUM(t.[MATUR] * t.balance_rub)
            / NULLIF(SUM(t.balance_rub), 0) AS matur

    FROM [ALM_TEST].[WORK].[vGroupDepositInterestsRate_For_FL] t WITH (NOLOCK)
    WHERE t.[Date] >= @DT_FROM
      AND t.[Date] <= @DT_TO
      AND t.[Признак опциональности для ФЛ] = 'all option_type'
      AND t.[Сегмент ФЛ] = 'all segment'
      AND t.[Срочн. бакеты] <> 'all termgroup'
      AND t.[Тип маржи] = 'all margin'
      AND t.[Сегмент бизнеса] = 'all segment'
      AND t.[Тип клиента] = 'all client'
      AND t.[Тип отчета] = 'Новые день ко дню'
      AND t.[Спред к КС бакеты] = 'all spread bucket'
      AND t.balance_rub IS NOT NULL
      AND t.balance_rub <> 0
      AND t.[MATUR] IS NOT NULL
      AND t.[MATUR] <> 0
    GROUP BY
        CAST(t.[Date] AS date)
),

v_fl_float AS (
    /* портфель плавающих вкладов ФЛ:
       оставляем как было,
       matur не выводим */
    SELECT
        'вклады ФЛ с плав ставкой' AS section_name,
        CAST(cal.[Date] AS date) AS dt_rep,
        'портфель' AS data_scope,

        SUM(ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])) AS out_rub_total,

        SUM(
            (
                kr.[KEY_RATE]
                + fl.[correction] / 100.0
                + ssv.[rate]
            ) * ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])
        ) / NULLIF(SUM(ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])), 0) AS rate_con,

        kr.[KEY_RATE] AS KEY_RATE,

        SUM(
            (
                fl.[correction] / 100.0
                + ssv.[rate]
            ) * ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])
        ) / NULLIF(SUM(ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])), 0) AS spread_keyrate,

        CAST(1 AS int) AS has_matur,
        CAST(NULL AS decimal(18, 6)) AS matur

    FROM [LIQUIDITY].[liq].[VW_FloatContracts] fl WITH (NOLOCK)

    JOIN [LIQUIDITY].[liq].[DepositContract_all] dep WITH (NOLOCK)
        ON fl.[con_id] = dep.[CON_ID]

    JOIN calendar cal
        ON cal.[Date] BETWEEN fl.[dt_from] AND fl.[dt_to]
       AND cal.[Date] >= dep.[DT_OPEN]
       AND cal.[Date] <  dep.[DT_CLOSE]

    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] sald WITH (NOLOCK)
        ON dep.[CON_ID] = sald.[CON_ID]
       AND cal.[Date] BETWEEN sald.[DT_FROM] AND sald.[DT_TO]

    LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache kr WITH (NOLOCK)
        ON cal.[Date] = kr.dt_rep
       AND kr.term = 1

    JOIN ALM.info.VW_SSVrates ssv WITH (NOLOCK)
        ON cal.[Date] BETWEEN ssv.[dt_from] AND ssv.[dt_to]

    WHERE LOWER(fl.[comment]) LIKE '%фл%'
      AND kr.[KEY_RATE] IS NOT NULL

    GROUP BY
        CAST(cal.[Date] AS date),
        kr.[KEY_RATE]


    UNION ALL


    /* новые плавающие вклады ФЛ:
       дата = дата открытия,
       matur = DATEDIFF(day, dep.DT_OPEN, dep.DT_CLOSE_PLAN),
       rate_con / KEY_RATE / spread_keyrate считаем с весом balance * matur,
       matur считаем с весом balance */
    SELECT
        'вклады ФЛ с плав ставкой' AS section_name,
        CAST(cal.[Date] AS date) AS dt_rep,
        'новые' AS data_scope,

        SUM(ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])) AS out_rub_total,

        SUM(
            (
                kr.[KEY_RATE]
                + fl.[correction] / 100.0
                + ssv.[rate]
            )
            * ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])
            * DATEDIFF(day, dep.[DT_OPEN], dep.[DT_CLOSE_PLAN])
        )
        / NULLIF(
            SUM(
                ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])
                * DATEDIFF(day, dep.[DT_OPEN], dep.[DT_CLOSE_PLAN])
            ),
            0
        ) AS rate_con,

        SUM(
            kr.[KEY_RATE]
            * ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])
            * DATEDIFF(day, dep.[DT_OPEN], dep.[DT_CLOSE_PLAN])
        )
        / NULLIF(
            SUM(
                ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])
                * DATEDIFF(day, dep.[DT_OPEN], dep.[DT_CLOSE_PLAN])
            ),
            0
        ) AS KEY_RATE,

        SUM(
            (
                fl.[correction] / 100.0
                + ssv.[rate]
            )
            * ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])
            * DATEDIFF(day, dep.[DT_OPEN], dep.[DT_CLOSE_PLAN])
        )
        / NULLIF(
            SUM(
                ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])
                * DATEDIFF(day, dep.[DT_OPEN], dep.[DT_CLOSE_PLAN])
            ),
            0
        ) AS spread_keyrate,

        CAST(1 AS int) AS has_matur,

        SUM(
            DATEDIFF(day, dep.[DT_OPEN], dep.[DT_CLOSE_PLAN])
            * ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])
        )
        / NULLIF(SUM(ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])), 0) AS matur

    FROM [LIQUIDITY].[liq].[VW_FloatContracts] fl WITH (NOLOCK)

    JOIN [LIQUIDITY].[liq].[DepositContract_all] dep WITH (NOLOCK)
        ON fl.[con_id] = dep.[CON_ID]

    JOIN calendar cal
        ON cal.[Date] = CAST(dep.[DT_OPEN] AS date)
       AND cal.[Date] BETWEEN fl.[dt_from] AND fl.[dt_to]

    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] sald WITH (NOLOCK)
        ON dep.[CON_ID] = sald.[CON_ID]
       AND cal.[Date] BETWEEN sald.[DT_FROM] AND sald.[DT_TO]

    LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache kr WITH (NOLOCK)
        ON cal.[Date] = kr.dt_rep
       AND kr.term = 1

    JOIN ALM.info.VW_SSVrates ssv WITH (NOLOCK)
        ON cal.[Date] BETWEEN ssv.[dt_from] AND ssv.[dt_to]

    WHERE LOWER(fl.[comment]) LIKE '%фл%'
      AND kr.[KEY_RATE] IS NOT NULL
      AND dep.[DT_CLOSE_PLAN] IS NOT NULL
      AND DATEDIFF(day, dep.[DT_OPEN], dep.[DT_CLOSE_PLAN]) <> 0

    GROUP BY
        CAST(cal.[Date] AS date)
)

SELECT
    section_name,
    dt_rep,
    data_scope,
    out_rub_total,
    rate_con,
    KEY_RATE,
    spread_keyrate,
    has_matur,
    matur
FROM DVS_FL

UNION ALL

SELECT
    section_name,
    dt_rep,
    data_scope,
    out_rub_total,
    rate_con,
    KEY_RATE,
    spread_keyrate,
    has_matur,
    matur
FROM v_fl_fix

UNION ALL

SELECT
    section_name,
    dt_rep,
    data_scope,
    out_rub_total,
    rate_con,
    KEY_RATE,
    spread_keyrate,
    has_matur,
    matur
FROM v_fl_float

ORDER BY
    dt_rep,
    section_name,
    data_scope;
