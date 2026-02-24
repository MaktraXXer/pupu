drop table cpr_report_new;

create table cpr_report_new as

with refin_rates as (
select last_day(to_date(to_char(dt_rep, 'YYYY-MM'), 'YYYY-MM')) period,
       round(avg(refin_rate), 4) refin_rate
from man_refin_rates -- Справочник с историческими ставками рефинансирования ипотеки на рынке
group by last_day(to_date(to_char(dt_rep, 'YYYY-MM'), 'YYYY-MM'))
),

/* Таблица исключений, развернутая в "платёжный период", который нужно исключить
   Логика: событие с датой excl_date исключаем в месяце payment_period = last_day(excl_date)
   (то есть для секьюритизации 2025-12-19 исключаем расчёт за payment_period = 2025-12-31) */
excl_by_payment_period as (
select e.con_id,
       last_day(e.excl_date) as payment_period,
       e.excl_reason
from cpr_exclusions e
where e.excl_reason = 'SECURITIZATION'
),

initial_table as ( -- Берём нужные признаки из mort_od, добавляем доп. переменные со сроками, группами по размеру ставки и т. д.
select con_id,
       dt_rep,
       dt_open_fact,
       dt_close_plan,
       dt_close_fact,
       round((dt_rep - dt_open_fact)/365, 2) as age,
       -out_rub as od,
       con_rate,
       agg_prod_name,
       segment_name,
       round((dt_close_plan - dt_rep)/365, 2) as term,
       (to_number(to_char(dt_close_plan, 'yyyy'), '9999') - to_number(to_char(dt_rep, 'yyyy'), '9999'))*12 +
                                              (to_number(to_char(dt_close_plan, 'mm'), '99') - to_number(to_char(dt_rep, 'mm'), '99')) term_months,
       case when con_rate < 0.03 then '0. <3%'
            when con_rate between 0.03 and 0.06 then '1. 3-6%'
            when con_rate between 0.06 and 0.09 then '2. 6-9%'
            when con_rate between 0.09 and 0.12 then '3. 9-12%'
            when con_rate between 0.12 and 0.15 then '4. 12-15%'
            when con_rate between 0.15 and 0.18 then '5. 15-18%'
            when con_rate between 0.18 and 0.21 then '6. 18-21%'
            when con_rate > 0.21 then '7. >21%'
            else '8. undefined rate' end rate_group,
       case when agg_prod_name in ('Семейная ипотека', 'Льготная ипотека', 'ИТ ипотека', 'Дальневосточная ипотека') then 1
            else 0 end as subsidy,
       last_day(dt_open_fact) as generation,
       refin_rate,
       round(100*(con_rate - refin_rate), 1) as stimul -- Стимул к рефинансированию
from mort_od
left join refin_rates
          on last_day(add_months(refin_rates.period, 1)) = mort_od.dt_rep
where 1 = 1
      and con_rate is not null
      and con_rate > 0
      and out_rub is not null
      and acc_role = 'DUE'
),

prelim_table as ( -- Добавляем группы по выдержке кредита, вычисляем аннуитетные платежи в разбивке на платёж процентов и платёж долга
select initial_table.*,
       case when age <= 1 then '0-1Y'
            when age between 1 and 2 then '1-2Y'
            when age between 2 and 3 then '2-3Y'
            when age between 3 and 4 then '3-4Y'
            when age between 4 and 5 then '4-5Y'
            when age between 5 and 6 then '5-6Y'
            when age between 6 and 7 then '6-7Y'
            when age between 7 and 8 then '7-8Y'
            when age between 8 and 9 then '8-9Y'
            when age > 9 then '9Y+'
            else 'undefined'
       end age_group,
       case when age <= 1 then 0
            when age between 1 and 2 then 1
            when age between 2 and 3 then 2
            when age between 3 and 4 then 3
            when age between 4 and 5 then 4
            when age between 5 and 6 then 5
            when age between 6 and 7 then 6
            when age between 7 and 8 then 7
            when age between 8 and 9 then 8
            else 9
       end age_group_id,
       case when term_months > 0 then round(od / (1 - power(1 + con_rate/12, -term_months)) * (con_rate/12), 2)
            else 0 end annuity,
       case when term_months > 0 then round(od * con_rate/12, 2)
            else 0 end interest_pay,
       case when term_months > 0 then round(od / (1 - power(1 + con_rate/12, -term_months)) * (con_rate/12), 2) - round(od * con_rate/12, 2)
            else 0 end od_plan_payment
from initial_table
),

final_bal as ( -- Для расчёта CPR нам нужен остаток долга ПОСЛЕ планового платежа долга
select prelim_table.*,
       od as od_before_plan,
       case when od - od_plan_payment < 0 then 0 else od - od_plan_payment end as od_after_plan
from prelim_table
),

pay as ( -- Собираем помесячные досрочные платежи по договорам
select con_id,
       last_day(dt_oper) dt,
       sum(vl_rub) premat_pay
from dds.con_oper
where 1 = 1
      and cur = '810'
      and oper_type = 'PAYMENT_PREMAT'
      and con_id in (select con_id from final_bal)
group by con_id, last_day(dt_oper)
)

select final_bal.*,
       last_day(add_months(final_bal.dt_rep, 1)) as payment_period,
       case when coalesce(pay.premat_pay, 0) >= od_after_plan then od_after_plan else coalesce(pay.premat_pay, 0) end premat_payment,
       params.b0 + params.b1 * ATAN(params.b2 + params.b3 * stimul) + params.b4 * ATAN(params.b5 + params.b6 * stimul) model_cpr -- Модельный CPR по модели Ценового центра
from final_bal
left join pay
          on 1 = 1
             and last_day(add_months(final_bal.dt_rep, 1)) = pay.dt
             and final_bal.con_id = pay.con_id
left join s_curve_params params -- Справочник с параметрами модели Ценового центра
          on final_bal.age_group_id = params.period
/* КЛЮЧЕВОЕ: исключаем con_id только в тот payment_period, когда была секьюритизация */
where not exists (
    select 1
    from excl_by_payment_period e
    where e.con_id = final_bal.con_id
      and e.payment_period = last_day(add_months(final_bal.dt_rep, 1))
)
order by final_bal.con_id,
         final_bal.dt_rep;
