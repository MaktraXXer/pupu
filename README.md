-- #1. Подготовка ключевой ставки
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

-- #2. Подготовка депозитов
DROP TABLE IF EXISTS #deposit;
SELECT *
INTO #deposit
FROM LIQUIDITY.liq.fnc_DepositContract_new(0,
           DATEADD(DAY, -365*5, CAST(GETDATE() AS date)),
           DATEADD(DAY,  365*5, CAST(GETDATE() AS date)),
           'NO_FL')
WHERE CLI_SHORT_NAME <> N'ФЛ';

;WITH d AS (
    SELECT
        dps.*,

        -- срок в днях (если DT_CLOSE_PLAN = 4444-01-01 или NULL -> 1)
        term_days = CASE
                        WHEN dps.DT_CLOSE_PLAN IS NULL
                          OR CAST(dps.DT_CLOSE_PLAN AS date) = DATEFROMPARTS(4444,1,1)
                             THEN 1
                        ELSE DATEDIFF(DAY, dps.DT_OPEN, dps.DT_CLOSE_PLAN)
                    END,

        -- конвенция до джойна в справочник
        base_convention = CASE
                              WHEN dps.CONVENTION = 'UNDEFINED'
                                   THEN CASE WHEN
                                             -- используем уже нормализованный term_days
                                             CASE
                                                 WHEN dps.DT_CLOSE_PLAN IS NULL
                                                   OR CAST(dps.DT_CLOSE_PLAN AS date) = DATEFROMPARTS(4444,1,1)
                                                      THEN 1
                                                 ELSE DATEDIFF(DAY, dps.DT_OPEN, dps.DT_CLOSE_PLAN)
                                             END
                                            <= 365
                                          THEN 'AT_THE_END'
                                          ELSE '1M'
                                     END
                              ELSE dps.CONVENTION
                         END
    FROM #deposit dps
)
SELECT
      d.*
    , man.IS_PDR

    -- Итоговая конвенция для расчётов: новая из справочника или исходная base_convention
    , Rate_Monthly = LIQUIDITY.liq.fnc_IntRate(
                        d.RATE,
                        ISNULL(conv.NEW_CONVENTION_NAME, d.base_convention),
                        'monthly',
                        d.term_days,
                        1
                      )
    , Spread2KeyRate = LIQUIDITY.liq.fnc_IntRate(
                          d.RATE,
                          ISNULL(conv.NEW_CONVENTION_NAME, d.base_convention),
                          'monthly',
                          d.term_days,
                          1
                        ) - kr.AVG_KEY_RATE
    , kr.KEY_RATE
    , MonthlyKeyRate = kr.AVG_KEY_RATE
FROM d
LEFT JOIN LIQUIDITY.liq.man_PROD_NAME_OPTIONLIAB man
       ON d.PROD_NAME = man.PROD_NAME
LEFT JOIN LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
       ON conv.CONVENTION_NAME = d.base_convention
LEFT JOIN #keyrate kr
       ON kr.TERM   = d.term_days
      AND kr.DT_REP = CAST(d.DT_OPEN AS date);
