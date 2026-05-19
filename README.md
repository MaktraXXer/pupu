DECLARE @DT_FROM date = '2025-01-01';
DECLARE @DT_TO   date = DATEADD(day, -2, CAST(GETDATE() AS date));

WITH calendar AS (
    SELECT 
        CAST([Date] AS date) AS [Date]
    FROM ALM.info.VW_calendar WITH (NOLOCK)
    WHERE [Date] BETWEEN @DT_FROM AND @DT_TO
),
keyrate_fact AS (
    SELECT
        CAST([date] AS date) AS [date],
        CAST([rate] AS float) / 100.0 AS keyrate
    FROM ALM.info.VW_CBKEY_everyday WITH (NOLOCK)
    WHERE [date] BETWEEN @DT_FROM AND @DT_TO
),
together AS (
    SELECT
        'НС' AS section_name,
        dt_rep,
        data_scope,
        out_rub_total,
        rate_con
    FROM ALM_TEST.mail.balance_metrics_savings WITH (NOLOCK)

    UNION ALL

    SELECT
        'ДВС' AS section_name,
        dt_rep,
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
        END AS rate_con
    FROM ALM_TEST.mail.balance_metrics_dvs WITH (NOLOCK)
),
DVS_FL AS (
    SELECT
        t.section_name,
        CAST(t.dt_rep AS date) AS dt_rep,
        t.data_scope,
        t.out_rub_total,
        t.rate_con + 0.0048 AS rate_con,
        kr.KEY_RATE,
        t.rate_con + 0.0048 - kr.KEY_RATE AS spread_keyrate
    FROM together t
    LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache kr WITH (NOLOCK)
        ON t.dt_rep = kr.dt_rep 
       AND kr.term = 1
    WHERE t.dt_rep >= @DT_FROM
),
v_fl_fix AS (
    SELECT 
        'вклады ФЛ с фикс ставкой' AS section_name,
        CAST(t.[Date] AS date) AS dt_rep,
        CASE 
            WHEN t.[Тип отчета] = 'Новые день ко дню' THEN 'новые'
            WHEN t.[Тип отчета] = 'Срез' THEN 'портфель'
        END AS data_scope,
        t.balance_rub AS out_rub_total,
        t.[MonthlyCONV_RATE_SSV] AS rate_con,
        t.[MonthlyCONV_ForecastKeyRate] AS KEY_RATE,
        t.[Spread_KeyRate] AS spread_keyrate
    FROM [ALM_TEST].[WORK].[vGroupDepositInterestsRate_For_FL] t WITH (NOLOCK)
    WHERE t.[Date] >= @DT_FROM
      AND t.[Date] <= @DT_TO
      AND t.[Признак опциональности для ФЛ] = 'all option_type'
      AND t.[Сегмент ФЛ] = 'all segment'
      AND t.[Срочн. бакеты] = 'all termgroup'
      AND t.[Тип маржи] = 'all margin'
      AND t.[Сегмент бизнеса] = 'all segment'
      AND t.[Тип клиента] = 'all client'
      AND t.[Тип отчета] IN ('Срез', 'Новые день ко дню')
      AND t.[Спред к КС бакеты] = 'all spread bucket'
),
v_fl_float AS (
    SELECT
        'вклады ФЛ с плав ставкой' AS section_name,
        CAST(cal.[Date] AS date) AS dt_rep,
        'портфель' AS data_scope,

        SUM(ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])) AS out_rub_total,

        SUM(
            (
                kr.[keyrate] 
                + fl.[correction] / 100.0 
                + ssv.[rate]
            ) * ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])
        ) / NULLIF(SUM(ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])), 0) AS rate_con,

        kr.[keyrate] AS KEY_RATE,

        SUM(
            (
                fl.[correction] / 100.0 
                + ssv.[rate]
            ) * ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])
        ) / NULLIF(SUM(ISNULL(sald.[OUT_RUB], dep.[BALANCE_RUB])), 0) AS spread_keyrate

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

    JOIN keyrate_fact kr
        ON cal.[Date] = kr.[date]

    JOIN ALM.info.VW_SSVrates ssv WITH (NOLOCK)
        ON cal.[Date] BETWEEN ssv.[dt_from] AND ssv.[dt_to]

    WHERE LOWER(fl.[comment]) LIKE '%фл%'

    GROUP BY 
        CAST(cal.[Date] AS date),
        kr.[keyrate]
)

SELECT
    section_name,
    dt_rep,
    data_scope,
    out_rub_total,
    rate_con,
    KEY_RATE,
    spread_keyrate
FROM DVS_FL

UNION ALL

SELECT
    section_name,
    dt_rep,
    data_scope,
    out_rub_total,
    rate_con,
    KEY_RATE,
    spread_keyrate
FROM v_fl_fix

UNION ALL

SELECT
    section_name,
    dt_rep,
    data_scope,
    out_rub_total,
    rate_con,
    KEY_RATE,
    spread_keyrate
FROM v_fl_float

ORDER BY
    dt_rep,
    section_name,
    data_scope;
