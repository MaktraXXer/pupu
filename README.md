/* ============================================================
   1) CPR только по секьюритизированному пулу (по каждому payment_period)
   Логика: договор относится к секьюритизации в тот payment_period,
   который равен last_day(excl_date).
   ============================================================ */
with excl_by_payment_period as (
    select e.con_id,
           last_day(e.excl_date) as payment_period
    from cpr_exclusions e
    where e.excl_reason = 'SECURITIZATION'
)
select
    r.payment_period as dt_rep,
    case when r.agg_prod_name is null then 'Ипотека Прочее' else r.agg_prod_name end as prod_name,
    sum(r.od_after_plan) as od_after_plan,
    sum(r.od) as od,
    sum(r.premat_payment) as premat,
    case
        when sum(r.od_after_plan) <= 0 then 0
        else round(100 * (1 - power(1 - sum(r.premat_payment)/sum(r.od_after_plan), 12)), 2)
    end as cpr
from cpr_report_new r
where r.payment_period between date'2024-01-01' and date'2025-12-31'
  and exists (
      select 1
      from excl_by_payment_period e
      where e.con_id = r.con_id
        and e.payment_period = r.payment_period
  )
group by
    r.payment_period,
    case when r.agg_prod_name is null then 'Ипотека Прочее' else r.agg_prod_name end
order by
    case when prod_name = 'Семейная ипотека' then 0
         when prod_name = 'Льготная ипотека' then 1
         when prod_name = 'ИТ ипотека' then 2
         when prod_name = 'Дальневосточная ипотека' then 3
         when prod_name = 'Первичная ипотека' then 4
         when prod_name = 'Вторичка' then 5
         when prod_name = 'Рефинансирование' then 6
         when prod_name = 'Военная ипотека' then 7
         when prod_name = 'ИЖС' then 8
         when prod_name = 'Ипотека Прочее' then 9
         else 10 end,
    r.payment_period;



/* ============================================================
   2) CPR по НЕсекьюритизированному пулу (по каждому payment_period)
   То есть "весь портфель МИНУС секьюритизированные con_id
   в соответствующий payment_period".
   ============================================================ */
with excl_by_payment_period as (
    select e.con_id,
           last_day(e.excl_date) as payment_period
    from cpr_exclusions e
    where e.excl_reason = 'SECURITIZATION'
)
select
    r.payment_period as dt_rep,
    case when r.agg_prod_name is null then 'Ипотека Прочее' else r.agg_prod_name end as prod_name,
    sum(r.od_after_plan) as od_after_plan,
    sum(r.od) as od,
    sum(r.premat_payment) as premat,
    case
        when sum(r.od_after_plan) <= 0 then 0
        else round(100 * (1 - power(1 - sum(r.premat_payment)/sum(r.od_after_plan), 12)), 2)
    end as cpr
from cpr_report_new r
where r.payment_period between date'2024-01-01' and date'2025-12-31'
  and not exists (
      select 1
      from excl_by_payment_period e
      where e.con_id = r.con_id
        and e.payment_period = r.payment_period
  )
group by
    r.payment_period,
    case when r.agg_prod_name is null then 'Ипотека Прочее' else r.agg_prod_name end
order by
    case when prod_name = 'Семейная ипотека' then 0
         when prod_name = 'Льготная ипотека' then 1
         when prod_name = 'ИТ ипотека' then 2
         when prod_name = 'Дальневосточная ипотека' then 3
         when prod_name = 'Первичная ипотека' then 4
         when prod_name = 'Вторичка' then 5
         when prod_name = 'Рефинансирование' then 6
         when prod_name = 'Военная ипотека' then 7
         when prod_name = 'ИЖС' then 8
         when prod_name = 'Ипотека Прочее' then 9
         else 10 end,
    r.payment_period;
