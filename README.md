/* ============================================================
   ШАГ 2. CPR_fact и CPR_model по 4 гос программам + итого
   для payment_period = 2026-01-31 и 2026-02-27

   Логика model:
   1) считаем CPR_model_con = b0 + b1*atan(b2+b3*stimul) + b4*atan(b5+b6*stimul)
      (это годовой CPR в долях)
   2) переводим в SMM_model = 1 - (1 - CPR_model_con)^(1/12)
   3) premat_model = od_after_plan * SMM_model
   4) CPR_model_portf = 1 - (1 - sum(premat_model)/sum(od_after_plan))^12
   ============================================================ */

with base as (
    select
        cpr.payment_period,
        cpr.agg_prod_name,
        cpr.age_group_id,
        cpr.stimul,
        cpr.od_after_plan,
        cpr.premat_payment,
        (s.b0 + s.b1 * ATAN(s.b2 + s.b3 * cpr.stimul) + s.b4 * ATAN(s.b5 + s.b6 * cpr.stimul)) as cpr_model_con
    from cpr_report_new cpr
    left join s_curve_params_gos s
           on cpr.age_group_id = s.period
    where 1 = 1
      and cpr.payment_period in (date'2026-01-31', date'2026-02-27')
      and cpr.stimul is not null
      and cpr.refin_rate > 0
      and cpr.con_rate > 0
      and cpr.agg_prod_name in ('Семейная ипотека','ИТ ипотека','Дальневосточная ипотека','Льготная ипотека')
),
base2 as (
    select
        payment_period,
        agg_prod_name,
        od_after_plan,
        premat_payment,
        /* premat_model из CPR_model_con */
        round(od_after_plan * (1 - power(1 - cpr_model_con, 1/12)), 2) as premat_model
    from base
)
select
    payment_period,
    case
        when grouping(agg_prod_name) = 1 then 'Все гос программы'
        else agg_prod_name
    end as prod_name,
    sum(od_after_plan) as od_after_plan,
    sum(premat_payment) as premat_fact,
    sum(premat_model) as premat_model,
    case
        when sum(od_after_plan) <= 0 then 0
        else round(100 * (1 - power(1 - sum(premat_payment)/sum(od_after_plan), 12)), 2)
    end as cpr_fact,
    case
        when sum(od_after_plan) <= 0 then 0
        else round(100 * (1 - power(1 - sum(premat_model)/sum(od_after_plan), 12)), 2)
    end as cpr_model
from base2
group by grouping sets (
    (payment_period, agg_prod_name),
    (payment_period)
)
order by
    payment_period,
    case
        when prod_name = 'Семейная ипотека' then 0
        when prod_name = 'Льготная ипотека' then 1
        when prod_name = 'ИТ ипотека' then 2
        when prod_name = 'Дальневосточная ипотека' then 3
        when prod_name = 'Все гос программы' then 9
        else 10
    end;
