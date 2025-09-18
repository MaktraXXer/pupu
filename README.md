-- 1) Ключевая
DROP TABLE IF EXISTS #keyrate;
SELECT DT_REP, TERM, KEY_RATE, AVG_KEY_RATE
INTO #keyrate
FROM ALM.info.VW_ForecastKEY_everyday WITH (NOLOCK)
WHERE DT_REP >= DATEADD(DAY, -365*5, CAST(GETDATE() AS date))
  AND TERM   <= 365;

-- 2) Депозиты
DROP TABLE IF EXISTS #deposit;
SELECT *
INTO #deposit
FROM LIQUIDITY.liq.fnc_DepositContract_new(
       0,
       DATEADD(DAY, -365*5, CAST(GETDATE() AS date)),
       DATEADD(DAY,  365*5, CAST(GETDATE() AS date)),
       'NO_FL'
     )
WHERE CLI_SHORT_NAME <> N'ФЛ';

-- 3) Итог без «лишних» полей
SELECT
      dps.*                                      -- только оригинальные поля депозита
    , man.IS_PDR
    , Rate_Monthly = LIQUIDITY.liq.fnc_IntRate(
                        dps.RATE,
                        ISNULL(conv.NEW_CONVENTION_NAME, norm_conv.norm_convention),
                        'monthly',
                        td.term_days,
                        1
                      )
    , Spread2KeyRate = LIQUIDITY.liq.fnc_IntRate(
                          dps.RATE,
                          ISNULL(conv.NEW_CONVENTION_NAME, norm_conv.norm_convention),
                          'monthly',
                          td.term_days,
                          1
                        ) - kr.AVG_KEY_RATE
    , kr.KEY_RATE
    , MonthlyKeyRate = kr.AVG_KEY_RATE
FROM #deposit dps

-- term_days: если DT_CLOSE_PLAN = '4444-01-01' или NULL -> 1, иначе DATEDIFF
CROSS APPLY (
    SELECT term_days = CASE
                           WHEN dps.DT_CLOSE_PLAN IS NULL
                             OR CAST(dps.DT_CLOSE_PLAN AS date) = DATEFROMPARTS(4444,1,1)
                             THEN 1
                           ELSE DATEDIFF(DAY, dps.DT_OPEN, dps.DT_CLOSE_PLAN)
                       END
) td

-- нормализованная конвенция для джойна/расчёта, но НЕ выводим её в SELECT
CROSS APPLY (
    SELECT norm_convention = CASE
        WHEN dps.CONVENTION = 'UNDEFINED'
             THEN CASE WHEN td.term_days <= 365 THEN 'AT_THE_END' ELSE '1M' END
        ELSE dps.CONVENTION
    END
) norm_conv

LEFT JOIN LIQUIDITY.liq.man_PROD_NAME_OPTIONLIAB man
       ON dps.PROD_NAME = man.PROD_NAME

LEFT JOIN LIQUIDITY.liq.man_CONVENTION conv WITH (NOLOCK)
       ON conv.CONVENTION_NAME = norm_conv.norm_convention

LEFT JOIN #keyrate kr
       ON kr.TERM   = td.term_days
      AND kr.DT_REP = CAST(dps.DT_OPEN AS date);
