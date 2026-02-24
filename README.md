/* ============================================================
   ТОП-5 сделок (con_id) по каждой программе по сумме досрочки (premat_payment)
   за декабрь 2025, только 4 программы:
   Семейная / Льготная / ИТ / Дальневосточная ипотека
   Только НЕсекьюритизированные.

   Дополнительно считаем CPR сделки (по её SMM):
     SMM = premat_payment / od_after_plan
     CPR = 100 * (1 - (1 - SMM)^12)
   ============================================================ */

with secur_cohort as (
    select distinct e.con_id
    from cpr_exclusions e
    where e.excl_reason = 'SECURITIZATION'
),

base as (
    select
        r.con_id,
        r.payment_period,
        r.agg_prod_name,
        r.segment_name,
        r.dt_rep,
        r.dt_open_fact,
        r.dt_close_plan,
        r.dt_close_fact,
        r.con_rate,
        r.refin_rate,
        r.stimul,
        r.age,
        r.term,
        r.term_months,
        r.od_after_plan,
        r.od,
        r.premat_payment,
        case
            when r.od_after_plan <= 0 then 0
            else round(100 * (1 - power(1 - (r.premat_payment / r.od_after_plan), 12)), 6)
        end as cpr_con
    from cpr_report_new r
    where r.payment_period = date'2025-12-31'
      and r.agg_prod_name in ('Семейная ипотека', 'Льготная ипотека', 'ИТ ипотека', 'Дальневосточная ипотека')
      and not exists (
          select 1
          from secur_cohort s
          where s.con_id = r.con_id
      )
),

ranked as (
    select
        b.*,
        row_number() over (
            partition by b.agg_prod_name
            order by b.premat_payment desc, b.od_after_plan desc, b.con_id
        ) as rn
    from base b
)

select
    con_id,
    agg_prod_name,
    segment_name,
    payment_period,
    dt_rep,
    dt_open_fact,
    dt_close_plan,
    dt_close_fact,
    con_rate,
    refin_rate,
    stimul,
    age,
    term,
    term_months,
    od_after_plan,
    premat_payment,
    cpr_con
from ranked
where rn <= 5
order by
    case when agg_prod_name = 'Семейная ипотека' then 0
         when agg_prod_name = 'Льготная ипотека' then 1
         when agg_prod_name = 'ИТ ипотека' then 2
         when agg_prod_name = 'Дальневосточная ипотека' then 3
         else 10 end,
    agg_prod_name,
    rn;
