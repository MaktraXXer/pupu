Понял. Делаем так, чтобы:
	1.	набор PROD_NAME не менялся из-за джоина (сначала отбираем «рыночные» продукты по имени, а не «через APPLY»),
	2.	надбавка PERCRATE всегда учитывалась, но если ставки для даты не нашли — берём 0 (чтобы строки не выпадали).

Ниже — финальная версия витрины с тем же набором колонок. Я заменил CROSS APPLY → OUTER APPLY, добавил явный фильтр по списку рыночных продуктов и COALESCE(ap.PERCRATE,0) в расчёте.

ALTER VIEW [WORK].[vDepositInterestsRate_FL_markets] AS
WITH mp AS (
    -- фиксируем список "рыночных" продуктов по имени, чтобы набор PROD_NAME не зависел от попадания в интервалы
    SELECT DISTINCT LTRIM(RTRIM(p.PROD_NAME)) AS PROD_NAME
    FROM [ALM_TEST].[markets].[prod_term_rates] p WITH (NOLOCK)
)
SELECT
    v.[DT_REP],
    v.[CON_ID],
    v.[CLI_ID],
    v.[INN],
    v.[CLI_SHORT_NAME],
    v.[IsDomRF],
    v.[ACC_NO],
    v.[PROD_TYPE],
    v.[DT_OPEN],
    v.[DT_CLOSE],
    v.[DT_CLOSE_PLAN],
    v.[MATUR],
    v.[RATE],
    v.[BALANCE_CUR],
    v.[CUR],
    v.[BALANCE_RUB],
    v.[CLI_SUBTYPE],
    v.[Group],
    v.[CONVENTION],
    v.[PROD_NAME],
    v.[IS_PDR],
    v.[IS_OPTION],
    v.[IS_FINANCE],
    v.[IS_FINANCE_LCR],
    v.[SEG_NAME],
    v.[LIQ_ФОР],
    v.[ALM_ФОР],
    v.[LIQ_ФОР_Fcast],
    v.[LIQ_ССВ],
    v.[ALM_ССВ],
    v.[LIQ_ССВ_Fcast],
    v.[ALM_LiquidityPrefRate],
    v.[ALM_OptionRate],
    v.[MonthlyCONV_RoisFix],
    v.[MonthlyCONV_LIQ_TransfertRate],
    v.[MonthlyCONV_ALM_TransfertRate],
    v.[MonthlyCONV_KBD],
    v.[MonthlyCONV_KRS],
    v.[MonthlyCONV_ForecastKeyRate],

    /* (AtTheEnd_RATE + PERCRATE) -> monthly; PERCRATE по умолчанию 0, чтобы строки не выпадали */
    CAST((
        SELECT [LIQUIDITY].[liq].[fnc_IntRate](
            v.[AtTheEnd_RATE] + COALESCE(ap.PERCRATE, 0.0),
            'at the end', 'monthly',
            v.[MATUR], 1
        )
    ) AS decimal(18,8)) AS [MonthlyCONV_RATE],

    v.[AtTheEnd_RoisFix],
    v.[AtTheEnd_LIQ_TransfertRate],
    v.[AtTheEnd_ALM_TransfertRate],
    v.[AtTheEnd_KBD],
    v.[AtTheEnd_KRS],
    v.[AtTheEnd_ForecastKeyRate],
    v.[AtTheEnd_RATE],
    v.[TSEGMENTNAME],
    v.[isfloat]
FROM [WORK].[vDepositInterestsRate_For_FL_only] v WITH (NOLOCK)
JOIN mp ON LTRIM(RTRIM(v.PROD_NAME)) = mp.PROD_NAME
OUTER APPLY (
    SELECT TOP (1) p.PERCRATE
    FROM [ALM_TEST].[markets].[prod_term_rates] p WITH (NOLOCK)
    WHERE LTRIM(RTRIM(p.PROD_NAME)) = mp.PROD_NAME
      AND v.[MATUR]   BETWEEN p.[TERMDAYS_FROM] AND p.[TERMDAYS_TO]
      AND v.[DT_OPEN] BETWEEN p.[DT_FROM]       AND p.[DT_TO]   -- при необходимости можно сменить на v.DT_REP
    ORDER BY p.[DT_FROM] DESC, p.[DT_TO] DESC, p.[TERMDAYS_FROM] DESC
) ap
WHERE v.[AtTheEnd_RATE] IS NOT NULL;
GO

Это гарантирует:
	•	Тот же набор PROD_NAME (все продукты из prod_term_rates, которые есть в исходных данных), включая длинные сроки (550+, 730+, 1100+ и т.д.).
	•	Надбавка учтена там, где нашлась; где не нашлась — использован 0, поэтому договора не теряются.

Быстрая сверка после EXEC WORK.prc_SaveSnap_Markets:

SELECT COUNT(*), SUM(BALANCE_RUB)
FROM WORK.snap_markets
WHERE '2025-07-01' BETWEEN DT_OPEN AND DT_CLOSE
  AND PROD_NAME IN (SELECT DISTINCT PROD_NAME FROM ALM_TEST.markets.prod_term_rates);

SELECT COUNT(*), SUM(BALANCE_RUB)
FROM WORK.DepositInterestsRateSnap
WHERE '2025-07-01' BETWEEN DT_OPEN AND DT_CLOSE
  AND PROD_NAME IN (SELECT DISTINCT PROD_NAME FROM ALM_TEST.markets.prod_term_rates);

Если захотите привязывать надбавку по дате отчёта (а не по дате открытия), замените условие в OUTER APPLY на v.DT_REP BETWEEN p.DT_FROM AND p.DT_TO.
