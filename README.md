DROP TABLE IF EXISTS #keyrate; SELECT DT_REP, TERM, KEY_RATE, AVG_KEY_RATE INTO #keyrate FROM ALM.info.VW_ForecastKEY_everyday WITH (NOLOCK) WHERE DT_REP >= GETDATE() - 365 * 5 AND TERM <= 365; DROP TABLE IF EXISTS #deposit; SELECT * INTO #deposit FROM LIQUIDITY.liq.fnc_DepositContract_new(0, GETDATE() - 365 * 5, GETDATE() + 365 * 5, 'NO_FL') WHERE CLI_SHORT_NAME <> 'ФЛ'; SELECT dps.*, man.IS_PDR, LIQUIDITY.liq.fnc_IntRate(dps.RATE, ISNULL(conv.NEW_CONVENTION_NAME, dps.CONVENTION), 'monthly', DATEDIFF(DAY, dps.dt_open, dps.dt_close_plan), 1) AS Rate_Monthly, LIQUIDITY.liq.fnc_IntRate(dps.RATE, ISNULL(conv.NEW_CONVENTION_NAME, dps.CONVENTION), 'monthly', DATEDIFF(DAY, dps.dt_open, dps.dt_close_plan), 1) - kr.AVG_KEY_RATE AS Spread2KeyRate, kr.KEY_RATE, kr.AVG_KEY_RATE AS MonthlyKeyRate FROM #deposit dps LEFT JOIN LIQUIDITY.liq.man_PROD_NAME_OPTIONLIAB man ON dps.PROD_NAME = man.PROD_NAME LEFT JOIN #keyrate kr ON DATEDIFF(DAY, dps.dt_open, dps.dt_close_plan) = kr.TERM AND dps.dt_open = kr.DT_REP LEFT JOIN LIQUIDITY.liq.man_CONVENTION conv WITH(NOLOCK) ON conv.CONVENTION_NAME = dps.CONVENTION;

помоги мне сделать так чтоб до джоина другого справочника конвенций в случае, когда установлена UNDEFINED, то мы ставим конвенцию (конвенцию к которой джоинится справочник по итогу с новыми конвенциями):
AT_THE_END – если , DATEDIFF(DAY, dps.dt_open, dps.dt_close_plan) <=365
1M  - , DATEDIFF(DAY, dps.dt_open, dps.dt_close_plan) >365
 
ТАКЖЕ ПОМОГИ УЧЕСТЬ ЧТОБЫ ВЕЗДЕ В ЗАПРОСЕ ЕСЛИ DT_CLOSE PLAN НЕОПРЕДЕЛЕН ТО СТАВИМ 1, НА ПРИМЕРЕ НИЖЕ ЗАПРОС ЧАСТЬ
 
case when dps.[DT_CLOSE_PLAN] = '4444-01-01' then 1 else datediff(DAY, dps.[DT_OPEN], dps.[DT_CLOSE_PLAN]) end ,
