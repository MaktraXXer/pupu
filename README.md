Ок, сделал «витрину» с пересчётом MonthlyCONV_RATE по правилам и снапшот под неё.

⸻

1) Витрина с учётом markets.prod_term_rates

(джоин по PROD_NAME, MATUR ∈ [TERMDAYS_FROM;TERMDAYS_TO], DT_OPEN ∈ [DT_FROM;DT_TO]; если совпадение нашлось — конвертируем MonthlyCONV_RATE → at the end → + PERCRATE → обратно в monthly по фактической CONVENTION; если совпадения нет — берём MonthlyCONV_RATE как есть)

USE [ALM_TEST];
GO
SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

CREATE OR ALTER VIEW [WORK].[vDepositInterestsRate_FL_markets]
AS
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

    /* --- Новый пересчитанный MonthlyCONV_RATE --- */
    CASE
        WHEN ap.PERCRATE IS NOT NULL THEN
            CAST((
                SELECT [LIQUIDITY].[liq].[fnc_IntRate](
                    /* сначала monthly -> at the end */
                    CAST((
                        SELECT [LIQUIDITY].[liq].[fnc_IntRate](
                            v.[MonthlyCONV_RATE],
                            'monthly', 'at the end',
                            v.[MATUR], 1
                        )
                    ) AS decimal(18,8))
                    + ap.PERCRATE,                -- прибавка из справочника (в at-the-end)
                    /* затем обратно: фактическая convention -> monthly */
                    v.[CONVENTION], 'monthly',
                    v.[MATUR], 1
                )
            ) AS decimal(18,8))
        ELSE
            v.[MonthlyCONV_RATE]
    END AS [MonthlyCONV_RATE],

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
OUTER APPLY (
    SELECT TOP (1) p.PERCRATE
    FROM [ALM_TEST].[markets].[prod_term_rates] p WITH (NOLOCK)
    WHERE LTRIM(RTRIM(p.PROD_NAME)) = LTRIM(RTRIM(v.PROD_NAME))
      AND v.[MATUR] BETWEEN p.[TERMDAYS_FROM] AND p.[TERMDAYS_TO]
      AND v.[DT_OPEN] BETWEEN p.[DT_FROM] AND p.[DT_TO]
    ORDER BY p.[DT_FROM] DESC, p.[DT_TO] DESC, p.[TERMDAYS_FROM] DESC
) ap;
GO

Примечание: базовая вьюха WORK.vDepositInterestsRate_For_FL_only уже фильтрует CLI_SUBTYPE='ФЛ', CUR='RUR' и берёт максимум DT_REP, так что отдельные проверки валюты/даты тут не требуются. Если диапазоны в prod_term_rates пересекаются, берём самую «свежую» запись по DT_FROM DESC.

⸻

2) Таблица для снапшота (snap_markets)

Самый простой и безошибочный способ — создать схему таблицы на основе витрины, чтобы столбцы и типы совпадали 1:1.

Одноразово выполните:

USE [ALM_TEST];
GO

IF OBJECT_ID('WORK.snap_markets','U') IS NULL
BEGIN
    SELECT TOP (0) *
    INTO WORK.snap_markets
    FROM WORK.vDepositInterestsRate_FL_markets;
END
GO

(Если хотите явный CREATE TABLE — скажите, выведу полный список столбцов с типами. Автогенерация от витрины минимизирует риск несовпадений при будущих правках.)

⸻

3) Процедура сохранения снапшота из витрины в WORK.snap_markets

Полностью аналогична вашей prc_SaveSnap, с логированием в WORK.Log_Events.

USE [ALM_TEST];
GO
SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

CREATE OR ALTER PROCEDURE [WORK].[prc_SaveSnap_Markets]
AS
BEGIN
    SET NOCOUNT ON;

    -- Если таблица ещё не создана (первый запуск) – создаём схему от витрины
    IF OBJECT_ID('WORK.snap_markets','U') IS NULL
    BEGIN
        SELECT TOP (0) *
        INTO WORK.snap_markets
        FROM WORK.vDepositInterestsRate_FL_markets;
    END

    -- Лог: старт
    INSERT INTO WORK.Log_Events ([Event], [EventDT], [User], [TableName], [ProcedureName])
    VALUES (N'Начато сохранение снимка (markets)', GETDATE(), SYSTEM_USER, N'snap_markets', N'prc_SaveSnap_Markets');

    -- Обновление данных
    TRUNCATE TABLE WORK.snap_markets;

    INSERT INTO WORK.snap_markets
    SELECT *
    FROM WORK.vDepositInterestsRate_FL_markets WITH (NOLOCK);

    -- Лог: финиш
    INSERT INTO WORK.Log_Events ([Event], [EventDT], [User], [TableName], [ProcedureName])
    VALUES (N'Завершено сохранение снимка (markets)', GETDATE(), SYSTEM_USER, N'snap_markets', N'prc_SaveSnap_Markets');
END;
GO


⸻

(опционально) Индекс для ускорения сопоставления

Рекомендую в markets.prod_term_rates добавить покрывающий индекс под используемые условия:

CREATE INDEX IX_prod_term_rates_match
ON [ALM_TEST].[markets].[prod_term_rates] (
    PROD_NAME, DT_FROM, DT_TO, TERMDAYS_FROM, TERMDAYS_TO
)
INCLUDE (PERCRATE);

Готово: витрина делает аккуратный OUTER APPLY (без раздувания строк), пересчитывает MonthlyCONV_RATE только при наличии матча, и снапшот сохраняет точную проекцию витрины.
