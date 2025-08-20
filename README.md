/****** Script for SelectTopNRows command from SSMS  ******/
SELECT TOP (1000) [id]
      ,[PROD_NAME]
      ,[PROD_ID]
      ,[TERMDAYS_FROM]
      ,[TERMDAYS_TO]
      ,[DT_FROM]
      ,[DT_TO]
      ,[PERCRATE]
      ,[load_dt]
  FROM [ALM_TEST].[markets].[prod_term_rates]
id	PROD_NAME	PROD_ID	TERMDAYS_FROM	TERMDAYS_TO	DT_FROM	DT_TO	PERCRATE	load_dt
1	Надёжный	2249	91	180	2024-09-11	2024-09-30	0.016000	2025-08-15 14:14:37.133
2	Надёжный Промо	3083	91	180	2024-09-11	2024-09-30	0.016000	2025-08-15 14:14:37.147
3	ДОМа надёжно	3082	91	180	2024-09-11	2024-09-30	0.010000	2025-08-15 14:14:37.150
4	Надёжный	2249	181	273	2024-09-11	2024-09-30	0.008000	2025-08-15 14:14:37.153

Эту табличку я сделал на основе выгрузки из экселя

Эксель на самом деле собирался на основе неудачной выгрузки от коллег в которой был запрос
with prod as
(
select '51' code, 'Надёжный' label from dual union all
select '52' code, 'Надёжный Промо' label from dual union all
select '53' code, 'ДОМа надёжно' label from dual union all
select '54' code, 'Надёжный старт' label from dual union all
select '55' code, 'Надёжный премиум' label from dual union all
select '56' code, 'Все в ДОМ' label from dual union all
select '57' code, 'Надёжный VIP' label from dual union all
select '60' code, 'Надёжный t2' label from dual union all
select '61' code, 'ПК sravni' label from dual union all
select '62' code, 'ПК banki' label from dual union all
select '63' code, 'ПК vbr' label from dual union all
select '91' code, 'Надёжный Мегафон' label from dual union all
select '92' code, 'Надёжный прайм' label from dual union all
select '93' code, 'Надёжный процент' label from dual


)

 select
 rv.ValidFromDate "начало действия"
, rv.ValidToDate-1 "окончание действия"
, trunc(rd.amountfrom) "срок от"
  ,trunc(rd.amountto)-1 "срок до"
  ,trunc(rt.amountfrom) "№ продукта"
  ,rt.fixrate
  ,rt.percrate
  ,prod.label
  from ods_001.ComputeRate crt, ods_001.CRateVersion rv, ods_001.CRateDependence rd, ods_001.CRateTable rt
  ,prod
  where    rv.Rate              = crt.Classified
--    and date'2025-07-31' -- дата    between rv.ValidFromDate and rv.ValidToDate-0.01
    and rd.RateVersion       = rv.Classified
--    and nDpnd          between rd.AmountFrom and rd.AmountTo-1
    and rt.RateVariant       = rd.Classified
--    and (nAmtFrom is null or nAmtFrom = rt.AmountFrom)
and crt.label ='% Клиентское вознаграждение (RUR)'
and crt.active_flag='Y'
and crt.deleted_flag='N'
and rv.active_flag='Y'
and rv.deleted_flag='N'
and rd.active_flag='Y'
and rd.deleted_flag='N'
and rt.active_flag='Y'
and rt.deleted_flag='N'
and prod.code=trunc(rt.amountfrom)
замечу что тут совсем не те же самые айдишники  что есть, я просто знаю какое им название в соответствии поставить
И по итогу мне нужен результат запроса ровно в том формате который я хочу
а не как в примере запроса выше
Для этого потребуется еще временная табличка
prod_name	PROD_ID
Надёжный прайм	3195
Надёжный процент	3194
Надёжный Промо	3083
Надёжный премиум	3081
ДОМа надёжно	3082
Надёжный Т2	3182
Надёжный Мегафон	3186
Надёжный старт	3155
Надёжный VIP	3080
Всё в ДОМ	3156
Надёжный	224


Перепишешь запрос?


![Uploading image.png…]()
