CREATE OR ALTER PROCEDURE liq.prc_Rebuild_DepositKeyrateSpread
    @KeyrateLookbackYears INT = 5,  -- лет назад по ключевой
    @DepositLookbackYears INT = 1,  -- депозиты: сколько лет назад
    @DepositForwardYears  INT = 1   -- депозиты: сколько лет вперёд (по плану)
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    BEGIN TRY
        /* === 1) temp: ключевая ставка (TERM <= 365) === */
        IF OBJECT_ID('tempdb..#keyrate') IS NOT NULL DROP TABLE #keyrate;
        SELECT
            DT_REP,
            TERM,
            KEY_RATE,
            AVG_KEY_RATE
        INTO #keyrate
        FROM ALM.info.VW_ForecastKEY_everyday WITH (NOLOCK)
        WHERE DT_REP >= DATEADD(YEAR, -@KeyrateLookbackYears, CAST(GETDATE() AS date))
          AND TERM   <= 365;

        /* === 2) temp: депозиты (окно -@DepositLookbackYears … +@DepositForwardYears), без ФЛ === */
        IF OBJECT_ID('tempdb..#deposit') IS NOT NULL DROP TABLE #deposit;
        SELECT *
        INTO #deposit
        FROM LIQUIDITY.liq.fnc_DepositContract_new (
                0,
                DATEADD(YEAR, -@DepositLookbackYears, CAST(GETDATE() AS date)),
                DATEADD(YEAR,  @DepositForwardYears,  CAST(GETDATE() AS date)),
                'NO_FL'
        )
        WHERE CLI_SHORT_NAME <> N'ФЛ';

        /* === 3) расчёты: MATUR, ставки, спреды === */
        WITH d AS (
            SELECT
                dps.*,
                CASE WHEN dps.DT_CLOSE_PLAN = '4444-01-01' THEN 1
                     ELSE DATEDIFF(DAY, dps.DT_OPEN, dps.DT_CLOSE_PLAN)
                END AS MATUR
            FROM #deposit dps
        ),
        base AS (
            SELECT
                d.*,
                man.IS_PDR,

                /* помесячная ставка по депозиту с учётом конвенции */
                LIQUIDITY.liq.fnc_IntRate(
                    d.RATE,
                    ISNULL(conv.NEW_CONVENTION_NAME, d.CONVENTION),
                    'monthly',
                    d.MATUR,
                    1
                ) AS Rate_Monthly,

                /* трансфертная ставка: годовая → помесячная (конвенция начисления) */
                CAST(
                    LIQUIDITY.liq.fnc_IntRate(
                        trf.Rate,
                        'AT_THE_END',     -- конвенция; при необходимости поставь свою
                        'monthly',
                        trf.Term,
                        1
                    ) AS DECIMAL(18,8)
                ) * (1 - fc.rate) AS MonthlyCONV_LIQ_TransfertRate,

                kr.KEY_RATE,
                kr.AVG_KEY_RATE AS MonthlyKeyRate,

                /* спред депозита к усреднённой «месячной» ключевой */
                LIQUIDITY.liq.fnc_IntRate(
                    d.RATE,
                    ISNULL(conv.NEW_CONVENTION_NAME, d.CONVENTION),
                    'monthly',
                    d.MATUR,
                    1
                ) - kr.AVG_KEY_RATE AS Spread2KeyRate
            FROM d
            LEFT JOIN LIQUIDITY.liq.man_PROD_NAME_OPTIONLIAB man
                   ON d.PROD_NAME = man.PROD_NAME
            LEFT JOIN #keyrate kr
                   ON (d.MATUR = kr.TERM)
                  AND (d.DT_OPEN = kr.DT_REP)
            LEFT JOIN [ALM].[info].[VW_FORrates_controlling] fc
                   ON (fc.CurName = 'RUR')
                  AND (d.DT_OPEN BETWEEN fc.dt_from AND DATEADD(DAY, -1, fc.dateNext))
            LEFT JOIN [ALM].[info].[VW_TransfertRates_everyday_settled_interpolated] trf
                   ON (d.DT_OPEN = trf.[Date])
                  AND (d.MATUR  = trf.[Term])
                  AND (trf.[Type] = 'BaseFixed')
                  AND (trf.[Currency] = 'RUB')
            LEFT JOIN LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
                   ON conv.CONVENTION_NAME = d.CONVENTION
        )

        /* === 4) целевая таблица: ВСЕГДА дропаем и создаём заново === */
        IF OBJECT_ID('liq.DepositKeyrateSpread','U') IS NOT NULL
            DROP TABLE liq.DepositKeyrateSpread;

        SELECT
            b.*
        INTO liq.DepositKeyrateSpread
        FROM base b;

        /* при необходимости — индексы (пример):
        -- CREATE INDEX IX_DepositKeyrateSpread_DT_OPEN ON liq.DepositKeyrateSpread(DT_OPEN);
        -- CREATE INDEX IX_DepositKeyrateSpread_PROD    ON liq.DepositKeyrateSpread(PROD_NAME);
        */

        SELECT COUNT(*) AS rows_loaded
        FROM liq.DepositKeyrateSpread;
    END TRY
    BEGIN CATCH
        DECLARE
            @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE(),
            @ErrNum INT = ERROR_NUMBER(),
            @ErrSev INT = ERROR_SEVERITY(),
            @ErrSta INT = ERROR_STATE(),
            @ErrLin INT = ERROR_LINE(),
            @ErrProc SYSNAME = ERROR_PROCEDURE();

        RAISERROR(N'[liq.prc_Rebuild_DepositKeyrateSpread] %s | Num=%d Sev=%d Sta=%d Line=%d Proc=%s',
                  @ErrSev, 1,
                  @ErrMsg, @ErrNum, @ErrSev, @ErrSta, @ErrLin, @ErrProc);
    END CATCH
END;
GO
