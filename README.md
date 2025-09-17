CREATE OR ALTER PROCEDURE liq.prc_Refresh_DepositKeyrateSpread
    @KeyrateLookbackYears INT = 5,   -- сколько лет назад брать ключ/кривые
    @DepositLookbackYears INT = 1,   -- сколько лет назад брать депозиты
    @DepositForwardYears  INT = 1,   -- сколько лет вперёд (по плану) брать депозиты
    @RebuildTarget        BIT = 0    -- 1 = дропнуть и пересоздать целевую таблицу
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    BEGIN TRY
        /* — опционально пересоздаём целевую таблицу */
        IF @RebuildTarget = 1 AND OBJECT_ID('liq.DepositKeyrateSpread','U') IS NOT NULL
            DROP TABLE liq.DepositKeyrateSpread;

        /* === 1) temp: ключевая ставка (до 365 дней срок), за @KeyrateLookbackYears === */
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

        /* === 2) temp: депозиты (окно: -@DepositLookbackYears ... +@DepositForwardYears), без ФЛ === */
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

                /* трансфертная ставка: годовая → помесячная (конвенция — AT_THE_END), на (1 - fc.rate) */
                CAST(
                    LIQUIDITY.liq.fnc_IntRate(
                        trf.Rate,
                        'AT_THE_END',     -- это конвенция начисления
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
                   ON d.MATUR   = kr.TERM
                  AND d.DT_OPEN = kr.DT_REP
            LEFT JOIN [ALM].[info].[VW_FORrates_controlling] fc
                   ON fc.CurName = 'RUR'
                  AND d.DT_OPEN BETWEEN fc.dt_from AND DATEADD(DAY, -1, fc.dateNext)
            LEFT JOIN [ALM].[info].[VW_TransfertRates_everyday_settled_interpolated] trf
                   ON d.DT_OPEN   = trf.[Date]
                  AND d.MATUR     = trf.[Term]
                  AND trf.[Type]  = 'BaseFixed'
                  AND trf.[Currency] = 'RUB'
            LEFT JOIN LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
                   ON conv.CONVENTION_NAME = d.CONVENTION
        )

        /* === 4) целевая таблица: создаём при первом запуске или очищаем и вставляем === */
        IF OBJECT_ID('liq.DepositKeyrateSpread','U') IS NULL
        BEGIN
            SELECT b.*
            INTO liq.DepositKeyrateSpread
            FROM base b;

            /* (необязательно) индексы — под свои частые фильтры:
            CREATE INDEX IX_DepositKeyrateSpread_DT_OPEN ON liq.DepositKeyrateSpread(DT_OPEN);
            CREATE INDEX IX_DepositKeyrateSpread_PROD ON liq.DepositKeyrateSpread(PROD_NAME);
            */
        END
        ELSE
        BEGIN
            TRUNCATE TABLE liq.DepositKeyrateSpread;

            INSERT INTO liq.DepositKeyrateSpread
            SELECT b.* FROM base b;
        END

        /* Возврат количества вставленных строк */
        SELECT COUNT(*) AS rows_loaded
        FROM liq.DepositKeyrateSpread;
    END TRY
    BEGIN CATCH
        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE(),
                @ErrNum INT = ERROR_NUMBER(),
                @ErrSev INT = ERROR_SEVERITY(),
                @ErrSta INT = ERROR_STATE(),
                @ErrLin INT = ERROR_LINE(),
                @ErrProc SYSNAME = ERROR_PROCEDURE();

        RAISERROR(N'[liq.prc_Refresh_DepositKeyrateSpread] %s | Num=%d Sev=%d Sta=%d Line=%d Proc=%s',
                  @ErrSev, 1,
                  @ErrMsg, @ErrNum, @ErrSev, @ErrSta, @ErrLin, @ErrProc);
    END CATCH
END;
GO
