USE [LIQUIDITY];
GO

IF OBJECT_ID('liq.prc_Rebuild_DepositKeyrateSpread','P') IS NOT NULL
    DROP PROCEDURE liq.prc_Rebuild_DepositKeyrateSpread;
GO

CREATE PROCEDURE liq.prc_Rebuild_DepositKeyrateSpread
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    BEGIN TRY
        BEGIN TRAN;

        /* ---------- 1) Источник ключевой ---------- */
        DROP TABLE IF EXISTS #keyrate;
        SELECT
            DT_REP,
            TERM,
            KEY_RATE,
            AVG_KEY_RATE
        INTO #keyrate
        FROM ALM.info.VW_ForecastKEY_everyday WITH (NOLOCK)
        WHERE DT_REP >= DATEADD(DAY, -365*5, CAST(GETDATE() AS date))
          AND TERM   <= 365;

        /* ---------- 2) Источник депозитов ---------- */
        DROP TABLE IF EXISTS #deposit;
        SELECT *
        INTO #deposit
        FROM LIQUIDITY.liq.fnc_DepositContract_new
             (0,
              DATEADD(DAY, -365*5, CAST(GETDATE() AS date)),
              DATEADD(DAY,  365*5, CAST(GETDATE() AS date)),
              'NO_FL')
        WHERE CLI_SHORT_NAME <> N'ФЛ';

        /* ---------- 3) Пересборка целевой таблицы ---------- */
        IF OBJECT_ID('liq.DepositKeyrateSpread','U') IS NOT NULL
            DROP TABLE liq.DepositKeyrateSpread;

        SELECT
            dps.*,
            man.IS_PDR,
            LIQUIDITY.liq.fnc_IntRate(
                dps.RATE,
                ISNULL(conv.NEW_CONVENTION_NAME, dps.CONVENTION),
                'monthly',
                DATEDIFF(DAY, dps.dt_open, dps.dt_close_plan),
                1
            ) AS Rate_Monthly,
            LIQUIDITY.liq.fnc_IntRate(
                dps.RATE,
                ISNULL(conv.NEW_CONVENTION_NAME, dps.CONVENTION),
                'monthly',
                DATEDIFF(DAY, dps.dt_open, dps.dt_close_plan),
                1
            ) - kr.AVG_KEY_RATE AS Spread2KeyRate,
            kr.KEY_RATE,
            kr.AVG_KEY_RATE AS MonthlyKeyRate
        INTO liq.DepositKeyrateSpread
        FROM #deposit dps
        LEFT JOIN LIQUIDITY.liq.man_PROD_NAME_OPTIONLIAB man
               ON dps.PROD_NAME = man.PROD_NAME
        LEFT JOIN #keyrate kr
               ON DATEDIFF(DAY, dps.dt_open, dps.dt_close_plan) = kr.TERM
              AND dps.dt_open = kr.DT_REP
        LEFT JOIN LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
               ON conv.CONVENTION_NAME = dps.CONVENTION;

        /* ---------- 4) Индексы (минимальный набор) ---------- */
        -- Подправь поля при необходимости, если их нет в dps.*
        BEGIN TRY
            CREATE CLUSTERED INDEX CX_DepositKeyrateSpread_dt
                ON liq.DepositKeyrateSpread (dt_open, dt_close_plan);
        END TRY BEGIN CATCH END CATCH;

        BEGIN TRY
            CREATE INDEX IX_DepositKeyrateSpread_prod
                ON liq.DepositKeyrateSpread (PROD_NAME);
        END TRY BEGIN CATCH END CATCH;

        BEGIN TRY
            CREATE INDEX IX_DepositKeyrateSpread_cli
                ON liq.DepositKeyrateSpread (CLI_ID);
        END TRY BEGIN CATCH END CATCH;

        COMMIT TRAN;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK TRAN;

        DECLARE
            @ErrMsg  nvarchar(4000) = ERROR_MESSAGE(),
            @ErrNum  int             = ERROR_NUMBER(),
            @ErrProc sysname         = ERROR_PROCEDURE(),
            @ErrLine int             = ERROR_LINE();

        RAISERROR(N'[liq.prc_Rebuild_DepositKeyrateSpread] %s (Num=%d, Proc=%s, Line=%d)',
                  16, 1, @ErrMsg, @ErrNum, @ErrProc, @ErrLine);
        RETURN;
    END CATCH
END
GO

-- Запуск:
EXEC liq.prc_Rebuild_DepositKeyrateSpread;
-- Проверка:
SELECT TOP (100) * FROM liq.DepositKeyrateSpread ORDER BY dt_open DESC;
