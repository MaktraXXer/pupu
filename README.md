Чтобы отследить, где именно теряются объемы при расчёте «Среза» ФЛ на 01.07, мы можем пошагово разбить процедуру и выполнить несколько промежуточных проверок. Ниже приведён план отладки:

⸻

🔍 Шаг 1. Посмотреть, какие сделки вообще попадают в DepositInterestsRateSnap на 01.07

Проверь объём всех ФЛ-сделок с ненулевыми остатками и трансфертной ставкой:

SELECT 
    COUNT(*) AS total_deals,
    SUM(saldo.OUT_RUB) AS total_rub
FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap] dep
JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] saldo 
    ON dep.CON_ID = saldo.CON_ID
    AND '2025-07-01' BETWEEN saldo.DT_FROM AND saldo.DT_TO
WHERE dep.DT_REP = (SELECT MAX(DT_REP) FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap])
  AND dep.CLI_SUBTYPE = 'ФЛ'
  AND ISNULL(dep.isfloat, 0) = 0
  AND dep.MonthlyCONV_ALM_TransfertRate IS NOT NULL
  AND saldo.OUT_RUB != 0

Если тут объём близок к 49 млрд — уже сужаем круг.

⸻

🔍 Шаг 2. Добавить построчно причины исключения сделок

Создай временную таблицу с диагностикой по фильтрам:

SELECT 
    dep.CON_ID,
    saldo.OUT_RUB,
    CASE WHEN dep.MonthlyCONV_ALM_TransfertRate IS NULL THEN 1 ELSE 0 END AS no_transf,
    CASE WHEN saldo.OUT_RUB = 0 THEN 1 ELSE 0 END AS zero_saldo,
    CASE WHEN dep.RATE <= 0.01 THEN 1 ELSE 0 END AS low_rate,
    CASE WHEN dep.MonthlyCONV_RATE IS NULL THEN 1 ELSE 0 END AS no_base_rate,
    CASE WHEN dep.LIQ_ФОР IS NULL THEN 1 ELSE 0 END AS no_liq_for,
    CASE WHEN dep.MonthlyCONV_ALM_TransfertRate - 
              (dep.MonthlyCONV_Rate + dep.LIQ_ССВ_Fcast + dep.ALM_OptionRate * dep.IS_OPTION)
             NOT BETWEEN -0.07 AND 0.07 THEN 1 ELSE 0 END AS spread_out_of_range
INTO #diag
FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap] dep
JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] saldo 
    ON dep.CON_ID = saldo.CON_ID
    AND '2025-07-01' BETWEEN saldo.DT_FROM AND saldo.DT_TO
WHERE dep.DT_REP = (SELECT MAX(DT_REP) FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap])
  AND dep.CLI_SUBTYPE = 'ФЛ'
  AND ISNULL(dep.isfloat, 0) = 0

Теперь можно агрегировать:

SELECT 
    COUNT(*) AS deals,
    SUM(OUT_RUB) AS rub,
    SUM(no_transf) AS null_transf,
    SUM(zero_saldo) AS zero_saldo,
    SUM(low_rate) AS low_rate,
    SUM(no_base_rate) AS null_base_rate,
    SUM(no_liq_for) AS null_liq_for,
    SUM(spread_out_of_range) AS bad_spread
FROM #diag


⸻

🔍 Шаг 3. Проверить фильтрацию calendar

Убедись, что дата 2025-07-01 точно попадает в #calendar. Бывали случаи, когда даты исчезали из VW_Calendar из-за глюков в источнике.

SELECT * 
FROM [ALM].[info].[VW_Calendar]
WHERE [Date] = '2025-07-01'


⸻

🔍 Шаг 4. Отдельно посчитать входящие сделки на 01.07 через i=1

Ты хочешь понять, сколько сделок ФЛ попадает в тип 'Срез':

SELECT COUNT(*) AS deals, SUM(saldo.OUT_RUB) AS rub
FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap] dep
JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] saldo ON dep.CON_ID = saldo.CON_ID
JOIN [ALM].[info].[VW_Calendar] cal ON dep.DT_OPEN <= cal.[Date] 
    AND dep.DT_CLOSE > cal.[Date]
WHERE cal.[Date] = '2025-07-01'
  AND dep.DT_REP = (SELECT MAX(DT_REP) FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap])
  AND dep.CLI_SUBTYPE = 'ФЛ'
  AND ISNULL(dep.isfloat, 0) = 0
  AND saldo.DT_FROM <= '2025-07-01' AND saldo.DT_TO >= '2025-07-01'


⸻

✔️ Резюме

Пошаговый план позволит выявить:
	1.	Все сделки, вообще имеющиеся в снапе с сальдо на 01.07;
	2.	Какие именно фильтры обнулили объёмы — и сколько;
	3.	Проблему с источником календаря;
	4.	Потерю типа 'Срез' по join-ам cal.Date.

Если хочешь, могу сразу прислать версию всей процедуры с дополнительным логированием в temp-таблицы на всех ключевых шагах.
