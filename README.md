/* ============================================================
   ТОП-20 ДОГОВОРОВ по сумме досрочки (premat_payment) за декабрь 2025
   Только НЕсекьюритизированные
   Только продукты: Семейная / Льготная / ИТ / Дальневосточная ипотека

   Примечание:
   - сортируем по premat_payment (убывание)
   - при равенстве можно добивать od_after_plan / con_id
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
        r.premat_payment
    from cpr_report_new r
    where r.payment_period = date'2025-12-31'
      and r.agg_prod_name in ('Семейная ипотека', 'Льготная ипотека', 'ИТ ипотека', 'Дальневосточная ипотека')
      and not exists (
          select 1
          from secur_cohort s
          where s.con_id = r.con_id
      )
)

select *
from (
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
        premat_payment
    from base
    order by premat_payment desc, od_after_plan desc, con_id
)
where rownum <= 20;
