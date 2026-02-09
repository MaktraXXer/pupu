USE [ALM_TEST]
GO

/****** Object:  StoredProcedure [WORK].[prc_GenerateGroupDepositInterestsRate_UL_matur_pdr_fo]    Script Date: 22.01.2026 9:58:35 ******/
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO
















/*
	Процедура формирует витрину со средневзвешенными спредами
	--------------------------------------------------------------------------------
	Vers.  Date         Developer       Description
	------ ----------- --------------- ---------------------------------------------
	02.01  2023-08-18   D.Smoyan        Created
	03.00  2025-03-04	M.Makhmud		Merged
	--------------------------------------------------------------------------------

*/




--ALTER PROCEDURE [liq].[prc_GenerateGroupDepositInterestsRate]
ALTER       PROCEDURE [WORK].[prc_GenerateGroupDepositInterestsRate_UL_matur_pdr_fo]
		@dt_Start date = null
		,@dt_End date = null
AS
BEGIN	
/*
<descr>
	Процедура обновляет витрину по средневзвешенным спредам
</descr>
*/

/*	/*>>ЛОГИ*/
		insert into LIQUIDITY.liq.log_events ([event], [eventdt], [user], [tablename], [seqid], [repdate], [criterianame], [criteriaval], [rowscount], [procedure_name])
		select 'начат расчет взвешенных спредов по ЮЛ' as event,getdate() as eventdt,user as [user],
		'liq.GroupDepositInterestsRate'	  as tablename,
		next value for liq.seq as seqid, null as repdate,
		null as criterianame,
		null as criteriaval
		, null as rowscount
		,'liq.prc_GenerateGroupDepositInterestsRate' as [procedure_name]
	/*<<ЛОГИ*/
*/

--насчет календаря подумай как лучше тебе сделать для автозапуска
	/*
	declare
		@dt_Start date = null --'2024-01-01'
		,@dt_End date = '2024-10-02'
	--*/
	/*set @dt_Start = isnull(@dt_Start, dateadd(month, datediff(month, 0, getdate()-30), 0))
	set @dt_End = isnull(@dt_End, (
		select max([DATE])
		from (
			select [DATE], datepart(WEEKDAY, [DATE]) week_day
			from [TradingDW].[dbo].[Calendar]
			where [Date] between  cast(getdate() - 7 as date) and cast(getdate() as date)
		) friday
		where week_day = 5
	)
	
	); */

--	set @dt_Start = isnull(@dt_Start,dateadd(day, -7, cast(getdate() as date)))
	set @dt_Start = isnull(@dt_Start,dateadd(day, -16, cast(getdate() as date)))
	set @dt_End = isnull(@dt_End, dateadd(day, -2, cast(getdate() as date)))

	drop table if exists #calendar
	select [Date]
	into #calendar
	from [ALM].[info].[VW_Calendar] with (nolock)
	where [Date] >= @dt_Start
		and [Date] <= @dt_End;

	--32m36s
	drop table if exists #depSpreads
	select
		case
			when i.[i] in (1, 4) then cal.[Date]
		--	when i.[i] = 2 then dateadd(day, -(datepart(weekday, dep.[DT_OPEN]) - 1), dep.[DT_OPEN])
		--	when i.[i] = 3 then dateadd(day, -(datepart(weekday, dep.[DT_CLOSE]) - 1), dep.[DT_CLOSE])
		--	when i.[i] = 5 then dateadd(month, datediff(month, 0, dep.[DT_OPEN]), 0)
			when i.[i] = 6 then dep.[DT_OPEN]
		end																									[Date]
		,case
			when i.[i] = 1 then 'Срез'
			--when i.[i] = 2 then 'Начало'
			--when i.[i] = 3 then 'Закрытие'
			--when i.[i] = 4 and dep.[MATUR_TYPE] = 'SHORT MATUR' then 'Срез коротких'
			--when i.[i] = 5 and dep.[CLI_SUBTYPE] != 'ФЛ' and dep.[MATUR_TYPE] = 'MEDIUM MATUR' then 'Начало среднесрочных'
			--when i.[i] = 5 and dep.[CLI_SUBTYPE] = 'ФЛ' then 'Начало среднесрочных'
			when i.[i]=6 /* and dep.[CLI_SUBTYPE] != 'ФЛ' */ then 'Начало день ко дню'
		end																									[TYPE]	
		,dep.[CLI_SUBTYPE]
		,isnull(dep.[MARGIN_TYPE], 'Прочий_тип')															[MARGIN_TYPE]
		,term.[TERM_GROUP]
		,cast(dep.[IS_OPTION] as varchar(255))																[IS_OPTION]
		,isnull(dep.[SEG_NAME], 'Прочий_сегмент')															[SEG_NAME]
		,cast(isnull(dep.[IS_PDR],0) as varchar(255))														[IS_PDR]
		,cast(isnull(dep.[IS_FINANCE_LCR],0) as varchar(255))													[IS_FINANCE_LCR]
		,sum(saldo.[OUT_RUB] *dep.[MATUR]) / sum(saldo.[OUT_RUB])											[MATUR]
		--, spr_buck.[Spread_Bucket]																								[Spread_KeyRate]
		,sum(saldo.[OUT_RUB])																				[BALANCE_RUB]				-- остаток в рублях
		,sum(saldo.[OUT_RUB]  * dep.[LIQ_ФОР]) / sum(saldo.[OUT_RUB])										[ФОР]						-- ФОР факстический по сделкам
		,sum(saldo.[OUT_RUB]  * dep.[LIQ_ССВ_Fcast]) / sum(saldo.[OUT_RUB])									[ССВ]						-- плата за ССВ, которая заложена в момент выдачи конкретной сделки
		,sum(saldo.[OUT_RUB]  * dep.[ALM_OptionRate]) / sum(saldo.[OUT_RUB])								[ALM_OptionRate]			-- плата за опциональность, которая заложена в момент выдачи конкретной сделки
		,sum(saldo.[OUT_RUB]  * dep.[LIQ_LiquidityRate]) / sum(saldo.[OUT_RUB])								[LIQ_LiquidityRate]			-- плата за опциональность, которая заложена в момент выдачи конкретной сделки
		,sum(saldo.[OUT_RUB]  *
			case
				when dep.[MATUR] <= 365 then dep.[MonthlyCONV_RoisFix]																	-- если срочность сделки меньше года, то используем значения  RoisFix, иначе КБД
				else dep.[MonthlyCONV_KBD]
			end
		) / sum(saldo.[OUT_RUB])																			[MonthlyCONV_OIS]
		--добавил расчет средневзвешенного  % к оттоку по нормативу ГВ
		,sum(saldo.[OUT_RUB]  *
			case
				when ([CLI_SUBTYPE] in ('ЮЛ Крупн.', 'ЮЛ ССВ')) and (isnull(dep.[IS_PDR],0)=1) then 0.4	
				when ([CLI_SUBTYPE] in ('ЮЛ Крупн.', 'ЮЛ ССВ')) and (isnull(dep.[IS_PDR],0)=0) then 0.4 * 70/ (0.5*(70+dep.[MATUR]+ABS(dep.[MATUR]-70)))
				else 0
			end
		) / sum(saldo.[OUT_RUB])																			[UL_OUTFLOW_LCR]

		,sum(saldo.[OUT_RUB]  *
			case
				when dep.[CLI_SUBTYPE] != 'ФЛ' then dep.[MonthlyCONV_LIQ_TransfertRate]
				else isnull(dep.[MonthlyCONV_ALM_TransfertRate], dep.[MonthlyCONV_LIQ_TransfertRate])
			end
		) / sum(saldo.[OUT_RUB])																			[MonthlyCONV_TransfertRate]
		,sum(saldo.[OUT_RUB]  *
			case
				when dep.[CLI_SUBTYPE] != 'ФЛ' then
					case
						when dep.[MonthlyCONV_LIQ_TransfertRate] - (dep.[MonthlyCONV_Rate] + dep.[LIQ_ССВ_Fcast]) >= 0 then dep.[MonthlyCONV_LIQ_TransfertRate]
						else (dep.[MonthlyCONV_Rate] + dep.[LIQ_ССВ_Fcast])
					end
				else isnull(dep.[MonthlyCONV_ALM_TransfertRate], dep.[MonthlyCONV_LIQ_TransfertRate])
			end
		) / sum(saldo.[OUT_RUB])																			[MonthlyCONV_TransfertRate_MOD]
		,sum(saldo.[OUT_RUB]  * dep.[MonthlyCONV_ALM_TransfertRate]) / sum(saldo.[OUT_RUB])					[MonthlyCONV_ALM_TransfertRate]
		,sum(saldo.[OUT_RUB]  * dep.[MonthlyCONV_KBD]) / sum(saldo.[OUT_RUB])								[MonthlyCONV_KBD]
		,sum(saldo.[OUT_RUB]  * isnull(dep.[MonthlyCONV_ForecastKeyRate], 0)) / sum(
			case
				when dep.[MonthlyCONV_ForecastKeyRate] is null then 0.000001
				else saldo.[OUT_RUB]  
			end
		)																									[MonthlyCONV_ForecastKeyRate]
		,sum(saldo.[OUT_RUB] * dep.[MonthlyCONV_Rate]) / sum(saldo.[OUT_RUB])								[MonthlyCONV_Rate]
	into #depSpreads
	from (
		select *
			,case
				when 	
					case
						when dep.[CLI_SUBTYPE] != 'ФЛ' then dep.[MonthlyCONV_LIQ_TransfertRate]
						else dep.[MonthlyCONV_ALM_TransfertRate]
					end - (dep.[MonthlyCONV_Rate] + dep.[LIQ_ССВ_Fcast]) < 0	then 'MINUS'
				when  	
					case
						when dep.[CLI_SUBTYPE] != 'ФЛ' then dep.[MonthlyCONV_LIQ_TransfertRate]
						else dep.[MonthlyCONV_ALM_TransfertRate]
					end - (dep.[MonthlyCONV_Rate] + dep.[LIQ_ССВ_Fcast]) >= 0	then 'PLUS'
			end		[MARGIN_TYPE]
			,case
				when dep.[MATUR] <= 31	then 'SHORT MATUR'
				when dep.[MATUR] between 32 and 181 then 'MEDIUM MATUR'
				when dep.[MATUR] >= 182 then 'LONG MATUR'
			end		[MATUR_TYPE]
		--	,dep.[MonthlyCONV_Rate] + dep.[LIQ_ССВ_Fcast] + dep.[ALM_OptionRate] - dep.[MonthlyCONV_ForecastKeyRate] [Spread_KeyRate]
		from [LIQUIDITY].[liq].[DepositInterestsRate] dep with (nolock)
		where [DT_REP] = (select max([DT_REP]) from [LIQUIDITY].[liq].[DepositInterestsRate] dep with (nolock))
			and case
				when dep.[CLI_SUBTYPE] != 'ФЛ' then dep.[MonthlyCONV_LIQ_TransfertRate]
				else isnull(dep.[MonthlyCONV_ALM_TransfertRate], dep.[MonthlyCONV_LIQ_TransfertRate])
			end is not null -- отсекаем сделки, ко которым трансферт не опредлен (в основном ФЛ)
			and isnull(dep.[isfloat], 0) = 0
			and dep.[CLI_SUBTYPE] in ('ЮЛ Крупн.', 'ЮЛ ССВ')
	) dep
		join (select 1 [i] union all select 2 union all select 3 union all select 4 union all select 5 union all select 6) i on 1 = 1
		join #calendar cal	on (
								dep.[DT_OPEN] <= cal.[Date]
								and dep.[DT_CLOSE] > cal.[Date]
								and i.[i] in (1, 4)
							)
							or (
								cal.[Date] = dep.[DT_OPEN]
								and i.[i] in (2, 5, 6)
							)
							or (
								cal.[Date] = dep.[DT_CLOSE]
								and i.[i] = 3
							)
							
		join [LIQUIDITY].[liq].[DepositContract_Saldo] saldo with (nolock)	on dep.[CON_ID] = saldo.[CON_ID]
																			and case
																				when i.[i] in (1, 4) then cal.[Date]
																				when i.[i] in (2, 5, 6) then dep.[DT_OPEN]
																				when i.[i] = 3 then dep.[DT_CLOSE_PLAN]
																			end
																			between saldo.[DT_FROM]	and saldo.[DT_TO]
		left join [LIQUIDITY].liq.[man_TermGroup] term with (nolock)		on dep.[MATUR] between term.[TERM_FROM] and term.[TERM_TO]
		--left join [LIQUIDITY].[liq].[man_Spread_Bucket] spr_buck with (nolock)		on dep.[Spread_KeyRate] > spr_buck.[Spread_From]
		--																			and dep.[Spread_KeyRate] <= spr_buck.[Spread_To]
	where 1 = 1
		and [CUR] = 'RUR'
		and [LIQ_ФОР] is not null
		and [MonthlyCONV_RATE] is not null
		and [IsDomRF] = 0
		and isnull(dep.[isfloat], 0) = 0 -- 28.12 П.Моргач
		and dep.[RATE] > 0.01
		and dep.[MonthlyCONV_RATE] between dep.[MonthlyCONV_ForecastKeyRate] - 0.07 and dep.[MonthlyCONV_ForecastKeyRate] + 0.07
		--and dep.[CON_ID] not in (select [CON_ID] from [LIQUIDITY].[liq].[man_FloatContracts] with (nolock)) -- исключаются сделки с плавающей ставкой
		and saldo.[OUT_RUB] != 0
		and dep.CLI_ID != 3731800 -- исключаем ФК
	--	and dep.con_id !=22771580
	group by
		case
			when i.[i] in (1, 4) then cal.[Date]
		--	when i.[i] = 2 then dateadd(day, -(datepart(weekday, dep.[DT_OPEN]) - 1), dep.[DT_OPEN])
		--	when i.[i] = 3 then dateadd(day, -(datepart(weekday, dep.[DT_CLOSE]) - 1), dep.[DT_CLOSE])
			--when i.[i] = 5 then dateadd(month, datediff(month, 0, dep.[DT_OPEN]), 0)
			when i.[i] = 6 then dep.[DT_OPEN]
		end																									
		,case
			when i.[i] = 1 then 'Срез'
		--	when i.[i] = 2 then 'Начало'
		--	when i.[i] = 3 then 'Закрытие'
		--	when i.[i] = 4 and dep.[MATUR_TYPE] = 'SHORT MATUR' then 'Срез коротких'
		--	when i.[i] = 5 and dep.[CLI_SUBTYPE] != 'ФЛ' and dep.[MATUR_TYPE] = 'MEDIUM MATUR' then 'Начало среднесрочных'
		--	when i.[i] = 5 and dep.[CLI_SUBTYPE] = 'ФЛ' then 'Начало среднесрочных'
			when i.[i] = 6 then 'Начало день ко дню'
		end			
		,dep.[CLI_SUBTYPE]
		,dep.[MARGIN_TYPE]
		,term.[TERM_GROUP]
		,dep.[IS_OPTION]
		,dep.[SEG_NAME]
		,dep.[IS_PDR]
		,dep.[IS_FINANCE_LCR]
		--,spr_buck.[Spread_Bucket]
		;
	/*	,case
			when i.[i] in (4, 5) and [Spread_KeyRate] <= 0 then '<= 0%'
			when i.[i] in (4, 5) and [Spread_KeyRate] > 0 and [Spread_KeyRate] <= 0.001 then '0% - 0.10%'
			when i.[i] in (4, 5) and [Spread_KeyRate] > 0.001 and [Spread_KeyRate] <= 0.002 then '0.10% - 0.20%'
			when i.[i] in (4, 5) and [Spread_KeyRate] > 0.002 and [Spread_KeyRate] <= 0.003 then '0.20% - 0.30%'
			when i.[i] in (4, 5) and [Spread_KeyRate] > 0.003 and [Spread_KeyRate] <= 0.004 then '0.30% - 0.40%'
			when i.[i] in (4, 5) and [Spread_KeyRate] > 0.004 and [Spread_KeyRate] <= 0.005 then '0.40% - 0.50%'
			when i.[i] in (4, 5) and [Spread_KeyRate] > 0.005 and [Spread_KeyRate] <= 0.006 then '0.50% - 0.60%'
			when i.[i] in (4, 5) and [Spread_KeyRate] > 0.006 and [Spread_KeyRate] <= 0.007 then '0.60% - 0.70%'
			when i.[i] in (4, 5) and [Spread_KeyRate] > 0.007 and [Spread_KeyRate] <= 0.008 then '0.70% - 0.80%'
			when i.[i] in (4, 5) and [Spread_KeyRate] > 0.008 and [Spread_KeyRate] <= 0.009 then '0.80% - 0.90%'
			when i.[i] in (4, 5) and [Spread_KeyRate] > 0.009 and [Spread_KeyRate] <= 0.010 then '0.90% - 1%'
			when i.[i] in (4, 5) and [Spread_KeyRate] > 0.01 then '> 1%'
		end	*/
	

	-- 3sek 208k
	drop table if exists #depSpreads2
	select
		cast([Date] as smalldatetime) [Date]
		,[TYPE]				
		,cast(CLI_SUBTYPE as varchar(255)) CLI_SUBTYPE
		,cast(MARGIN_TYPE as varchar(255)) MARGIN_TYPE
		,TERM_GROUP
		,IS_OPTION
		,IS_PDR
		,[IS_FINANCE_LCR]
		,SEG_NAME
	--	,cast(Spread_KeyRate as varchar(255)) Spread_KeyRate
		,sum([BALANCE_RUB])															[BALANCE_RUB]
		,SUM([BALANCE_RUB]*[UL_OUTFLOW_LCR])/SUM([BALANCE_RUB]) AS [UL_OUTFLOW_LCR]
		,SUM([BALANCE_RUB] *[MATUR])  / SUM([BALANCE_RUB])   AS [MATUR]
		,sum([BALANCE_RUB] * [ФОР]) / sum([BALANCE_RUB])							[ФОР]
		,sum([BALANCE_RUB] * [ССВ]) / sum([BALANCE_RUB])							[ССВ]
		,sum([BALANCE_RUB] * [ALM_OptionRate]) / sum([BALANCE_RUB])					[ALM_OptionRate]
		,sum([BALANCE_RUB] * [LIQ_LiquidityRate]) / sum([BALANCE_RUB])				[LIQ_LiquidityRate]
		,sum([BALANCE_RUB] * [MonthlyCONV_OIS]) / sum([BALANCE_RUB])				[MonthlyCONV_OIS]
		,sum([BALANCE_RUB] * [MonthlyCONV_TransfertRate]) / sum([BALANCE_RUB])		[MonthlyCONV_TransfertRate]
		,sum([BALANCE_RUB] * [MonthlyCONV_TransfertRate_MOD]) / sum([BALANCE_RUB])	[MonthlyCONV_TransfertRate_MOD]
		,sum([BALANCE_RUB] * [MonthlyCONV_ALM_TransfertRate]) / sum([BALANCE_RUB])	[MonthlyCONV_ALM_TransfertRate]
		,sum([BALANCE_RUB] * [MonthlyCONV_KBD]) / sum([BALANCE_RUB])				[MonthlyCONV_KBD]
		,sum([BALANCE_RUB] * [MonthlyCONV_ForecastKeyRate]) / sum([BALANCE_RUB])	[MonthlyCONV_ForecastKeyRate]
		,sum([BALANCE_RUB] * [MonthlyCONV_Rate]) / sum([BALANCE_RUB])				[MonthlyCONV_Rate]
	into #depSpreads2
	from #depSpreads
	group by cube(
		[Date]
		,[TYPE]				
		,CLI_SUBTYPE
		,MARGIN_TYPE
		,TERM_GROUP
		,IS_OPTION
		,IS_PDR
		,[IS_FINANCE_LCR]
		,SEG_NAME
		--,Spread_KeyRate
		)
	union all
	select *
	from #depSpreads
	UNION ALL
	SELECT
	  cast([Date] as smalldatetime) [Date]
     ,[TYPE]
     ,'ЮЛ'
     ,cast(MARGIN_TYPE as varchar(255)) MARGIN_TYPE
     ,[TERM_GROUP]
     ,[IS_OPTION]
	  ,[IS_PDR]
	  ,[IS_FINANCE_LCR]
     ,[SEG_NAME]
   --  ,cast(Spread_KeyRate as varchar(255)) Spread_KeyRate
     ,sum([BALANCE_RUB])															[BALANCE_RUB]
	  ,SUM([BALANCE_RUB]*[UL_OUTFLOW_LCR])/SUM([BALANCE_RUB]) AS [UL_OUTFLOW_LCR]
	  ,SUM([BALANCE_RUB] *[MATUR])  / SUM([BALANCE_RUB])   AS [MATUR]
	  ,sum([BALANCE_RUB] * [ФОР]) / sum([BALANCE_RUB])								[ФОР]
	  ,sum([BALANCE_RUB] * [ССВ]) / sum([BALANCE_RUB])								[ССВ]
	  ,sum([BALANCE_RUB] * [ALM_OptionRate]) / sum([BALANCE_RUB])					[ALM_OptionRate]
	  ,sum([BALANCE_RUB] * [LIQ_LiquidityRate]) / sum([BALANCE_RUB])				[LIQ_LiquidityRate]
	  ,sum([BALANCE_RUB] * [MonthlyCONV_OIS]) / sum([BALANCE_RUB])					[MonthlyCONV_OIS]
	  ,sum([BALANCE_RUB] * [MonthlyCONV_TransfertRate]) / sum([BALANCE_RUB])		[MonthlyCONV_TransfertRate]
	  ,sum([BALANCE_RUB] * [MonthlyCONV_TransfertRate_MOD]) / sum([BALANCE_RUB])	[MonthlyCONV_TransfertRate_MOD]
	  ,sum([BALANCE_RUB] * [MonthlyCONV_ALM_TransfertRate]) / sum([BALANCE_RUB])	[MonthlyCONV_ALM_TransfertRate]
	  ,sum([BALANCE_RUB] * [MonthlyCONV_KBD]) / sum([BALANCE_RUB])					[MonthlyCONV_KBD]
	  ,sum([BALANCE_RUB] * [MonthlyCONV_ForecastKeyRate]) / sum([BALANCE_RUB])		[MonthlyCONV_ForecastKeyRate]
	  ,sum([BALANCE_RUB] * [MonthlyCONV_Rate]) / sum([BALANCE_RUB])					[MonthlyCONV_Rate]
	 FROM #depSpreads
	 WHERE [CLI_SUBTYPE] in ('ЮЛ Крупн.', 'ЮЛ ССВ')
	 GROUP BY CUBE ([Date]
			  ,[TYPE]
			  ,MARGIN_TYPE
			  ,TERM_GROUP
			  ,IS_OPTION
			  ,SEG_NAME
			  ,[IS_FINANCE_LCR]
			  ,IS_PDR
			--  ,Spread_KeyRate
			  );

	with t_for_transf as (
	select *, row_number() over(partition by [Date]
		,[TYPE]				
		,CLI_SUBTYPE
		,MARGIN_TYPE
		,TERM_GROUP
		,IS_OPTION
		,SEG_NAME
		,IS_PDR
		,[IS_FINANCE_LCR]
	--	,Spread_KeyRate
		,BALANCE_RUB
		,[UL_OUTFLOW_LCR]
		,MATUR
		,[ФОР]
		,[ССВ]
		,[ALM_OptionRate]
		,[LIQ_LiquidityRate]
		,[MonthlyCONV_OIS]
		,[MonthlyCONV_TransfertRate]
		,[MonthlyCONV_TransfertRate_MOD]
		,[MonthlyCONV_ALM_TransfertRate]
		,[MonthlyCONV_KBD]
		,[MonthlyCONV_ForecastKeyRate]
		,[MonthlyCONV_Rate]
		order by [Date]
		) rn1
	from #depSpreads2)

	select * into #t_for_transf
	from t_for_transf
	where rn1 = 1

	delete from #t_for_transf
	where 1=1
	and [Type] is null

	delete from #t_for_transf
	where 1=1
	and [Date] is null

	--select * from #t_for_transf

	update #t_for_transf
	set
	TERM_GROUP = isnull(TERM_GROUP, 'all termgroup'),
	IS_OPTION = isnull(IS_OPTION, 'all option_type'),
	SEG_NAME = isnull(SEG_NAME, 'all segment'),
	CLI_SUBTYPE = isnull(CLI_SUBTYPE, 'all client'),
	MARGIN_TYPE = isnull(MARGIN_TYPE, 'all margin'),
	IS_PDR = isnull(IS_PDR,'all PDR'),
	IS_FINANCE_LCR = isnull(IS_FINANCE_LCR, 'all FINANCE_LCR')
	--Spread_KeyRate = isnull(Spread_KeyRate, 'all spread bucket')

	--delete from LIQUIDITY.liq.GroupDepositInterestsRate_dtStart
	--delete from liq.GroupDepositInterestsRate
	delete from [WORK].[GroupDepositInterestsRate_UL_matur_pdr_fo]
	where [Date] >= @dt_Start
	--(select min([Date]) from #depSpreads2);

	--insert into liq.GroupDepositInterestsRate
	insert into [WORK].[GroupDepositInterestsRate_UL_matur_pdr_fo]
	select [Date]
	,[TYPE]				
	,CLI_SUBTYPE
	,MARGIN_TYPE
	,TERM_GROUP
	,IS_OPTION
	,SEG_NAME
	,IS_PDR
	,IS_FINANCE_LCR
	--,Spread_KeyRate
	,BALANCE_RUB
	,[UL_OUTFLOW_LCR]
	,MATUR
	,[ФОР]
	,[ССВ]
	,[ALM_OptionRate]
	,[MonthlyCONV_OIS]
	,[MonthlyCONV_TransfertRate]
	,[MonthlyCONV_TransfertRate_MOD]
	,[MonthlyCONV_ALM_TransfertRate]
	,[MonthlyCONV_KBD]
	,[MonthlyCONV_ForecastKeyRate]
	,[MonthlyCONV_Rate]
	,getdate()
	,[LIQ_LiquidityRate]
	from #t_for_transf
	where 1 = 1
		and [TYPE] is not null
		and [CLI_SUBTYPE] is not null
		and [MARGIN_TYPE] is not null
	--	and [Spread_KeyRate] is not null

	drop table if exists #t_for_transf

	/*
	/*>>ЛОГИ*/
		insert into LIQUIDITY.liq.log_events ([event], [eventdt], [user], [tablename], [seqid], [repdate], [criterianame], [criteriaval], [rowscount], [procedure_name])
		select 'завершен расчет взвешенных спредов по ЮЛ' as event,getdate() as eventdt,user as [user],
		'liq.GroupDepositInterestsRate'		as tablename,
		next value for liq.seq as seqid, null as repdate,

		null as criterianame,
		null as criteriaval
		, null as rowscount
		,'liq.prc_GenerateGroupDepositInterestsRate' as [procedure_name]
	/*<<ЛОГИ*/
	*/

/*	drop table if exists #DynamicFinanceLCR_Deposit
	select
		dt.[Date]
		,dep.[IS_FINANCE_LCR]
		,sum(dep.[BALANCE_RUB]) [BALANCE_RUB]
		,getdate() [DT_CALC]
	into #DynamicFinanceLCR_Deposit
	from LIQUIDITY.liq.DepositInterestsRate dep with (nolock)
		join #calendar dt on dt.[Date] >= dep.[DT_OPEN]
					and dt.[Date] < dep.[DT_CLOSE]
	where dep.[DT_REP] = (select max([DT_REP]) from [LIQUIDITY].[liq].[DepositInterestsRate] with (nolock))
		and [IsDomRF] = 0
		and [CLI_SUBTYPE] != 'ФЛ'
	group by dt.[Date]
		,dep.[IS_FINANCE_LCR];

	delete from LIQUIDITY.liq.DynamicFinanceLCR_Deposit;
	insert into LIQUIDITY.liq.DynamicFinanceLCR_Deposit
	select * from #DynamicFinanceLCR_Deposit;

	truncate table [LIQUIDITY].[qt].[man_OIS_x_KeyRate];
	with ois as (
		select *
		from [ALM].[info].[VW_OIS_everyday]
		where [RateDate] = (
			select max([Date])
			from ALM.info.[VW_calendar]
			where [Date] <= cast(getdate() - 2 as date)
				and [IsWorkDay] = 1
			)
	)
	, kr as (
		select *
		from ALM.info.VW_ForecastKEY_everyday
		where DT_REP = (
			select max([Date])
			from ALM.info.[VW_calendar]
			where [Date] <= cast(getdate() - 2 as date)
				and [IsWorkDay] = 1
		)
	)
	insert into [LIQUIDITY].[qt].[man_OIS_x_KeyRate]
	select ois.[RateDate]
		,ois.[Term]
		,term.[TERM_GROUP_SHORT_NAME_NEW]  [SettlementCode]
		,(select [LIQUIDITY].[liq].[fnc_IntRate](ois.[Rate], 'annual', 'monthly', ois.[Term], 1)) [OIS]
		,kr.[AVG_KEY_RATE] [KeyRate]
		,kr.[AVG_KEY_RATE] - (select [LIQUIDITY].[liq].[fnc_IntRate](ois.[Rate], 'annual', 'monthly', ois.[Term], 1)) [Spread]
	from ois
		inner hash join kr on ois.[Term] = kr.[TERM]
		join [LIQUIDITY].[liq].[man_TermGroup_For_Quotation] term with (nolock) on ois.[Term] = term.[TERM_AVG]
	where term.[TERM_GROUP_SHORT_NAME_NEW] not in ('3W', '9M');

	*/
end;
GO

то есть в такой процедуре все срезы корректные?


Нет. В текущем виде срезы в витрине не гарантированно корректные, потому что у тебя в #depSpreads2 есть критичная ошибка:

1) UNION ALL select * from #depSpreads ломает поля местами

#depSpreads2 создаётся с порядком:

... IS_OPTION, IS_PDR, IS_FINANCE_LCR, SEG_NAME, BALANCE_RUB, ...

а #depSpreads имеет порядок:

... IS_OPTION, SEG_NAME, IS_PDR, IS_FINANCE_LCR, MATUR, BALANCE_RUB, ...

Поэтому в строках, которые попадают во вторую часть UNION ALL select *, SEG_NAME попадает в IS_PDR, IS_PDR — в IS_FINANCE_LCR и т.д. Отсюда и “PDR как категории сегмента”. И это влияет на любые срезы/агрегации, где участвует IS_PDR.

✅ Исправление: убрать select * и явно перечислить колонки в том же порядке, что в первом SELECT into #depSpreads2.

⸻

2) По “типа срезов” (TYPE) у тебя фактически остаются только 2 вида

Ты join-ишь i = 1..6, но:
	•	TYPE заполняется только для i=1 ('Срез') и i=6 ('Начало день ко дню')
	•	для i=4 (и остальных) TYPE = NULL → потом ты их удаляешь:

delete from #t_for_transf where [Type] is null

То есть в финал реально уходят только:
	•	Срез (ежедневный, по календарю, i=1)
	•	Начало день ко дню (DT_OPEN, i=6)

Если ты ожидал ещё “закрытие/начало/короткие” — сейчас их нет (они закомментированы / обнуляются).

⸻

Минимальный фикс (главное)

Заменить:

union all
select *
from #depSpreads

на явный список колонок в порядке #depSpreads2:

union all
select
    cast([Date] as smalldatetime) as [Date],
    [TYPE],
    cast(CLI_SUBTYPE as varchar(255)) as CLI_SUBTYPE,
    cast(MARGIN_TYPE as varchar(255)) as MARGIN_TYPE,
    TERM_GROUP,
    IS_OPTION,
    IS_PDR,
    [IS_FINANCE_LCR],
    SEG_NAME,
    [BALANCE_RUB],
    [UL_OUTFLOW_LCR],
    [MATUR],
    [ФОР],
    [ССВ],
    [ALM_OptionRate],
    [LIQ_LiquidityRate],
    [MonthlyCONV_OIS],
    [MonthlyCONV_TransfertRate],
    [MonthlyCONV_TransfertRate_MOD],
    [MonthlyCONV_ALM_TransfertRate],
    [MonthlyCONV_KBD],
    [MonthlyCONV_ForecastKeyRate],
    [MonthlyCONV_Rate]
from #depSpreads

После этого “PDR как SEG_NAME” исчезнет, и срезы по IS_PDR станут корректными.

Если хочешь — я сразу перепишу весь блок формирования #depSpreads2 (включая третий UNION ALL на 'ЮЛ') так, чтобы во всех частях был один порядок столбцов и не было риска повторения.
