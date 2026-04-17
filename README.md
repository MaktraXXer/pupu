/* ============================================================
   CPR_fact и CPR_model по ВСЕМ гос программам вместе
   в разрезе бакетов ставки + строка "ВСЕ БАКЕТЫ СТАВКИ"
   ДЛЯ КАЖДОГО payment_period в диапазоне 2024-01-01 .. 2026-02-27

   "ВСЕ БАКЕТЫ СТАВКИ" = агрегат по всем bucket_num внутри payment_period.
   ============================================================ */

with
enriched as (
    select
        cpr.payment_period,
        cpr.od_after_plan,
        cpr.od,
        cpr.premat_payment,
        cpr.con_rate,
        cpr.refin_rate,
        cpr.stimul,
        cpr.age_group_id,
        greatest(cpr.con_rate * 100, 0) as pct,
        case
            when greatest(cpr.con_rate * 100, 0) < 8.75 then
                floor((greatest(cpr.con_rate * 100, 0) + 0.25) / 0.5) + 1
            when greatest(cpr.con_rate * 100, 0) < 10 then 19
            when greatest(cpr.con_rate * 100, 0) < 12 then 20
            when greatest(cpr.con_rate * 100, 0) < 15 then 21
            when greatest(cpr.con_rate * 100, 0) < 20 then 22
            else 23
        end as bucket_num
    from cpr_report_new cpr
    where 1 = 1
      and cpr.payment_period between date'2024-01-01' and date'2026-02-27'
      and cpr.agg_prod_name in ('Семейная ипотека','ИТ ипотека','Дальневосточная ипотека','Льготная ипотека')
      and cpr.stimul is not null
      and cpr.refin_rate > 0
      and cpr.con_rate > 0
),

bucket_names as (
    select 1 as bucket_num,  'A. 0% (0.0-0.25%)'   as bucket_name from dual union all
    select 2,  'B. 0.5% (0.25-0.75%)' from dual union all
    select 3,  'C. 1.0% (0.75-1.25%)' from dual union all
    select 4,  'D. 1.5% (1.25-1.75%)' from dual union all
    select 5,  'E. 2.0% (1.75-2.25%)' from dual union all
    select 6,  'F. 2.5% (2.25-2.75%)' from dual union all
    select 7,  'G. 3.0% (2.75-3.25%)' from dual union all
    select 8,  'H. 3.5% (3.25-3.75%)' from dual union all
    select 9,  'I. 4.0% (3.75-4.25%)' from dual union all
    select 10, 'J. 4.5% (4.25-4.75%)' from dual union all
    select 11, 'K. 5.0% (4.75-5.25%)' from dual union all
    select 12, 'L. 5.5% (5.25-5.75%)' from dual union all
    select 13, 'M. 6.0% (5.75-6.25%)' from dual union all
    select 14, 'N. 6.5% (6.25-6.75%)' from dual union all
    select 15, 'O. 7.0% (6.75-7.25%)' from dual union all
    select 16, 'P. 7.5% (7.25-7.75%)' from dual union all
    select 17, 'Q. 8.0% (7.75-8.25%)' from dual union all
    select 18, 'R. 8.5% (8.25-8.75%)' from dual union all
    select 19, 'S. 8.75-10.0%' from dual union all
    select 20, 'T. 10.0-12.0%' from dual union all
    select 21, 'U. 12.0-15.0%' from dual union all
    select 22, 'V. 15.0-20.0%' from dual union all
    select 23, 'W. 20.0%+' from dual
),

base_with_model as (
    select
        e.payment_period,
        e.bucket_num,
        e.od_after_plan,
        e.od,
        e.premat_payment,
        round(
            e.od_after_plan
            * (1 - power(
                    1 - (s.b0 + s.b1 * atan(s.b2 + s.b3 * e.stimul) + s.b4 * atan(s.b5 + s.b6 * e.stimul)),
                    1/12
                )),
            2
        ) as premat_model
    from enriched e
    left join s_curve_params_gos s
           on e.age_group_id = s.period
)

select
    b.payment_period as dt_rep,
    case
        when grouping(b.bucket_num) = 1 then 'ВСЕ БАКЕТЫ СТАВКИ'
        else bn.bucket_name
    end as bucket_name,
    sum(b.od_after_plan) as od_after_plan,
    sum(b.od) as od,
    sum(b.premat_payment) as premat_fact,
    sum(b.premat_model) as premat_model,
    case
        when sum(b.od_after_plan) <= 0 then 0
        else round(100 * (1 - power(1 - sum(b.premat_payment)/sum(b.od_after_plan), 12)), 2)
    end as cpr_fact,
    case
        when sum(b.od_after_plan) <= 0 then 0
        else round(100 * (1 - power(1 - sum(b.premat_model)/sum(b.od_after_plan), 12)), 2)
    end as cpr_model
from base_with_model b
left join bucket_names bn
       on b.bucket_num = bn.bucket_num
group by grouping sets (
    (b.payment_period, b.bucket_num, bn.bucket_name),
    (b.payment_period)
)
order by
    b.payment_period,
    case when bucket_name = 'ВСЕ БАКЕТЫ СТАВКИ' then 0 else 1 end,
    b.bucket_num;
