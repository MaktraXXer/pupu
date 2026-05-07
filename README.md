declare @dt_start date = '20260101';
declare @dt_end   date = '20260503';

drop table if exists #keyrate;

select 
      [date]
    , [rate] / 100.0 [rate]
into #keyrate
from ALM.info.VW_CBKEY_everyday 
where [Date] between @dt_start and @dt_end;


drop table if exists #ruonia;

select 
      [date]
    , [rate] / 100.0 [rate]
into #ruonia
from ALM.info.VW_Ruonia_everyday
where [Date] between @dt_start and @dt_end;


drop table if exists #calendar;

select *
into #calendar
from ALM.info.[VW_calendar]
where [Date] between @dt_start and @dt_end;


drop table if exists #deposit;

select cal.[Date]
                ,dep.[CON_ID]
                ,dep.[ACC_NO]
                ,dep.[CLI_ID]
                ,dep.[INN]
                ,case
                                when CLI_ID in ('3731800', '3939590185') then 'Фед.Казна'
                                else dep.[CLI_SHORT_NAME]
                end                        [CLI_SHORT_NAME]
                ,dep.[BALANCE_CUR]
                ,dep.[CUR]
                ,saldo.[OUT_RUB]            [BALANCE_RUB]
                ,dep.[DT_OPEN]
                ,dep.[DT_CLOSE]
                ,dep.[DT_CLOSE_PLAN]
                ,dep.[MATUR]
                ,datediff(day, cal.[date], dep.[DT_CLOSE_PLAN]) [DURATION]
                ,dep.[SEG_NAME]
                ,dep.[PROD_NAME]
                ,dep.[IS_PDR]
                ,case 
                                when isnull(man.[basis], man2.[basis]) is null then 0 
                                else 1 
                end                                                                 [IS_FLOAT]
                ,dep.[OKVED_CODE]
                ,dep.[OKVED_NAME]
                ,dep.[SSVRate_Fcast]
                ,case     
                                when isnull(man.[basis], man2.[basis]) is not null then 
                                                (kr.[rate] / (1 - 0.0450) - kr.[rate])
                                else dep.[RFRate_Fcast_Int]
                end                                                                 [FOR_Rate]
                ,isnull(man.[basis], man2.[basis])                                  [BASIS]
                ,case 
                                when isnull(man.[basis], man2.[basis]) is not null then rn.[rate] 
                end                                                                 [BASIS_RATE]
                ,case 
                                when  isnull(man.[basis], man2.[basis])  is not null then isnull(man.[con_spread], man2.correction) / 100.0 
                end                                                                 [BASIS_SPREAD]
                ,case     
                                when isnull(man.[basis], man2.[basis]) is not null then 
                                                (rn.[rate] + isnull(man.[con_spread], man2.correction) / 100.0 
                                                                - (kr.[rate] / (1 - 0.0450) - kr.[rate]))
                                else dep.[RATE]
                end                                                                 [RATE]
                ,case     
                                when isnull(man.[basis], man2.[basis]) is not null then 
                                                (select liq.fnc_IntRate((rn.[rate] + isnull(man.[con_spread], man2.correction) / 100.0
                                                                - (kr.[rate] / (1 - 0.0450) - kr.[rate])), 'at the end', 'monthly', dep.[MATUR], 1)
                                                ) 
                                else [MonthlyConv_RATE]
                end                                                                 [MonthlyConv_Rate]
                ,dep.[KeyRate]
                ,dep.[MonthlyConv_OISRate]
                ,case     
                                when isnull(man.[basis], man2.[basis]) is not null then 
                                                (select liq.fnc_IntRate((rn.[rate] + isnull(man.[con_spread], man2.correction) / 100.0
                                                                - (kr.[rate] / (1 - 0.0450) - kr.[rate])), 'at the end', 'monthly', dep.[MATUR], 1)
                                                ) 
                                else dep.[MonthlyConv_RATE] + dep.[SSVRate_Fcast]
                end - dep.[KeyRate]                                                 [Spread_KeyRate]
                ,case     
                                when isnull(man.[basis], man2.[basis]) is not null then 
                                                (select liq.fnc_IntRate((rn.[rate] + isnull(man.[con_spread], man2.correction) / 100.0
                                                                - (kr.[rate] / (1 - 0.0450) - kr.[rate])), 'at the end', 'monthly', dep.[MATUR], 1)
                                                ) 
                                else MonthlyConv_RATE + dep.[SSVRate_Fcast]
                end - [MonthlyConv_OISRate]                                         [Spread_OIS]
                ,case
                                when dep.TransfertRate * (1 - [RFRate_Controlling]) + isnull(lr.[Value], 0) - (case
                                when isnull(man.[basis], man2.[basis]) is not null then 
                                                (rn.[rate] + isnull(man.[con_spread], man2.correction) / 100.0 
                                                                - (kr.[rate] / (1 - 0.0450) - kr.[rate]))
                                else dep.[RATE]
                end        + dep.SSVRate_Fcast) <= 0 then 0 
                                else dep.TransfertRate * (1 - [RFRate_Controlling]) + isnull(lr.[Value], 0) - (case   
                                when isnull(man.[basis], man2.[basis]) is not null then 
                                                (rn.[rate] + isnull(man.[con_spread], man2.correction) / 100.0 
                                                                - (kr.[rate] / (1 - 0.0450) - kr.[rate]))
                                else dep.[RATE]
                end        + dep.SSVRate_Fcast) 
                end [Margin]
                ,case
                                when intr.[RFRate_TRF] is not null then intr.[BaseRate_TRF] + isnull(intr.[LiquidityPrefRate_TRF], 0) + intr.[RFRate_TRF] + isnull(intr.OptionRate_TRF, 0) + coalesce(intr.[liqrate_TRF], lr.[Value], 0)
                                else (intr.[BaseRate_TRF] + isnull(intr.[LiquidityPrefRate_TRF], 0)) * (1 - dep.[RFRate_Controlling]) + isnull(intr.OptionRate_TRF, 0) + coalesce(intr.[liqrate_TRF], lr.[Value], 0)
                end [TransfertRate_Controlling]
                ,dep.TransfertRate * (1 - [RFRate_Controlling]) + isnull(lr.[Value], 0) [TransfertRate]
                ,case
                                when 
                                                case
                                                                when intr.[RFRate_TRF] is not null then intr.[BaseRate_TRF] + isnull(intr.[LiquidityPrefRate_TRF], 0) + intr.[RFRate_TRF] + isnull(intr.OptionRate_TRF, 0) + coalesce(intr.[liqrate_TRF], lr.[Value], 0)
                                                                else (intr.[BaseRate_TRF] + isnull(intr.[LiquidityPrefRate_TRF], 0)) * (1 - dep.[RFRate_Controlling]) + isnull(intr.OptionRate_TRF, 0) + coalesce(intr.[liqrate_TRF], lr.[Value], 0)
                                                end - (case          
                                when isnull(man.[basis], man2.[basis]) is not null then 
                                                (rn.[rate] + isnull(man.[con_spread], man2.correction) / 100.0 
                                                                - (kr.[rate] / (1 - 0.0450) - kr.[rate]))
                                else dep.[RATE]
                end        + dep.SSVRate_Fcast) <= 0 then 0 
                                else
                                                case
                                                                when intr.[RFRate_TRF] is not null then intr.[BaseRate_TRF] + isnull(intr.[LiquidityPrefRate_TRF], 0) + intr.[RFRate_TRF] + isnull(intr.OptionRate_TRF, 0) + coalesce(intr.[liqrate_TRF], lr.[Value], 0)
                                                                else (intr.[BaseRate_TRF] + isnull(intr.[LiquidityPrefRate_TRF], 0)) * (1 - dep.[RFRate_Controlling]) + isnull(intr.OptionRate_TRF, 0) + coalesce(intr.[liqrate_TRF], lr.[Value], 0)
                                                end - (case          
                                when isnull(man.[basis], man2.[basis]) is not null then 
                                                (rn.[rate] + isnull(man.[con_spread], man2.correction) / 100.0 
                                                                - (kr.[rate] / (1 - 0.0450) - kr.[rate]))
                                else dep.[RATE]
                end        + dep.SSVRate_Fcast) 
                end [Margin_Controlling]
into #deposit
from [LIQUIDITY].rep.DepositContract_LEGAL_InterestsRate dep with (nolock)
                join #calendar cal 
                                on dep.[DT_OPEN] <= cal.[Date]
                               and dep.[DT_CLOSE] > cal.[Date]

                join [LIQUIDITY].[liq].[DepositContract_Saldo] saldo with (nolock)
                                on dep.[CON_ID] = saldo.[CON_ID]
                               and cal.[Date] between saldo.[DT_FROM] and saldo.[DT_TO]

                left join LIQUIDITY.liq.floatrate_deals man 
                                on dep.CON_ID = cast(man.[con_id] as varchar(255))

                left join LIQUIDITY.liq.man_FloatContracts man2 
                                on cast(man2.[con_id] as varchar(255)) = dep.[con_id]

                left join #keyrate kr 
                                on cal.[Date] = kr.[date]

                left join #ruonia rn 
                                on cal.[Date] = rn.[date]

                left join alm.[info].[VW_liquidity_rates_interpolated] lr with (nolock) 
                                on dep.DT_OPEN between lr.[dt_from] and lr.[dt_to]
                               and lr.[Cur] = 810
                               and dep.[CUR] = 'RUR'
                               and isnull(dep.[IS_PDR], 0) = lr.[IS_PDR]
                               and isnull(dep.[IS_FINANCE_LCR], 0) = lr.[IS_FINANCE_LCR]
                               and dep.[MATUR] = lr.[Term]

                left join LIQUIDITY.liq.InterestsRateForDeposit intr with (nolock) 
                                on dep.[CON_ID] = intr.[CON_ID]

where dep.DT_CLOSE > @dt_start
--              and dep.[SEG_NAME] = 'Казна'
                and dep.ISDOMRF = 0
                and dep.[CUR] = 'RUR'
                and saldo.[OUT_RUB] is not null
                and saldo.[OUT_RUB] <> 0;


select *
                ,[Spread_KeyRate] * [BALANCE_RUB]                    [Spread_KeyRate_x_BalanceRub]
                ,[Spread_OIS] * [BALANCE_RUB]                        [Spread_OIS_x_BalanceRub]
                ,[DURATION] * [BALANCE_RUB]                          [DURATION_x_BalanceRub]
                ,[MATUR] * [BALANCE_RUB]                             [MATUR_x_BalanceRub]
                ,case
                                when isnull([Margin_Controlling], [Margin]) = 0 then ([RATE] + [SSVRate_Fcast])
                                else isnull([TransfertRate_Controlling], [TransfertRate]) 
                end - [KeyRate]                                      [Spread_KeyRate_TR]
                ,(case
                                when isnull([Margin_Controlling], [Margin]) = 0 then ([RATE] + [SSVRate_Fcast])
                                else isnull([TransfertRate_Controlling], [TransfertRate]) 
                end - [KeyRate]) * [BALANCE_RUB]                     [Spread_KeyRate_TR_x_BalanceRub]
                ,eomonth([date])                                     [Period]
from #deposit
where abs(Spread_KeyRate) <= 0.08;
