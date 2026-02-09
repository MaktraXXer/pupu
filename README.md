insert into [WORK].[GroupDepositInterestsRate_UL_matur_pdr_fo] (
   [Date],[TYPE],CLI_SUBTYPE,MARGIN_TYPE,TERM_GROUP,IS_OPTION,SEG_NAME,IS_PDR,IS_FINANCE_LCR,
   BALANCE_RUB,UL_OUTFLOW_LCR,MATUR,[ФОР],[ССВ],[ALM_OptionRate],
   [MonthlyCONV_OIS],[MonthlyCONV_TransfertRate],[MonthlyCONV_TransfertRate_MOD],
   [MonthlyCONV_ALM_TransfertRate],[MonthlyCONV_KBD],[MonthlyCONV_ForecastKeyRate],[MonthlyCONV_Rate],
   DT_CALC, [LIQ_LiquidityRate]
)
select
   [Date],[TYPE],CLI_SUBTYPE,MARGIN_TYPE,TERM_GROUP,IS_OPTION,SEG_NAME,IS_PDR,IS_FINANCE_LCR,
   BALANCE_RUB,UL_OUTFLOW_LCR,MATUR,[ФОР],[ССВ],[ALM_OptionRate],
   [MonthlyCONV_OIS],[MonthlyCONV_TransfertRate],[MonthlyCONV_TransfertRate_MOD],
   [MonthlyCONV_ALM_TransfertRate],[MonthlyCONV_KBD],[MonthlyCONV_ForecastKeyRate],[MonthlyCONV_Rate],
   getdate(), [LIQ_LiquidityRate]
from #t_for_transf
...
