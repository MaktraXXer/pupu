
 
 
И вот скриптики
Скрипт, по которому я считал модельные и фактические досрочные погашения:
select age_group_id, stimul, sum(od_after_plan) sum_od, sum(premat_payment) sum_premat_fact, sum(premat_model) sum_premat_model from ( select t.*, round(od_after_plan * (1 - power(1 - cpr_model, 1/12)), 2) premat_model from ( select cpr.*, case when od_after_plan <= 0 then 0 else round(100*(1 - power(1 - premat_payment / od_after_plan, 12)), 2)/100 end cpr_fact, round(100*(s.b0 + s.b1 * ATAN(s.b2 + s.b3 * stimul) + s.b4 * ATAN(s.b5 + s.b6 * stimul)), 2)/100 as cpr_model from cpr_report_new cpr left join s_curve_params s on cpr.age_group_id = s.period where 1 = 1 and stimul is not null and refin_rate > 0 and con_rate > 0 and cpr.age_group_id <= 2 --and agg_prod_name in ('Семейная ипотека','ИТ ипотека','Дальневосточная ипотека') and agg_prod_name = 'Семейная ипотека' ) t ) t1 group by age_group_id, stimul order by age_group_id, stimul
 
select AGG_PROD_NAME, age_group_id as LoanAge, date'2025-08-01' as dat, stimul as Incentive, case when sum(od_after_plan) <= 0 then 0 else round(100 * (1 - power(1 - sum(premat_payment)/sum(od_after_plan), 12)), 2) end as CPR, sum(od_after_plan) from cpr_report_new where 1 = 1 and stimul is not null and refin_rate > 0 and con_rate > 0 and AGG_PROD_NAME IN ('Вторичка','Первичная ипотека') group by AGG_PROD_NAME,age_group_id, stimul order by age_group_id,AGG_PROD_NAME, stimul
 
 
