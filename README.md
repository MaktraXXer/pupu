/* ============================================================
   1) Историческая CPR-динамика по "когорте" секьюритизированных con_id
      (те же con_id, которые потом секьюритизировали), по каждому payment_period
   Важно: НЕ привязываем к месяцу секьюритизации, просто берём список con_id.
   При желании можно сузить по конкретной дате секьюритизации (см. комментарий).
   ============================================================ */

with secur_cohort as (
    select distinct e.con_id
    from cpr_exclusions e
    where e.excl_reason = 'SECURITIZATION'
    -- если нужно строго под сделку 19.12.2025, раскомментируй:
    -- and e.excl_date = date'2025-12-19'
)
select
    r.payment_period as dt_rep,
    case when r.agg_prod_name is null then 'Ипотека Прочее' else r.agg_prod_name end as prod_name,
    sum(r.od_after_plan) as od_after_plan,
    sum(r.od) as od,
    sum(r.premat_payment) as premat,
    case when sum(r.od_after_plan) <= 0 then 0
         else round(100 * (1 - power(1 - sum(r.premat_payment)/sum(r.od_after_plan), 12)), 2)
    end as CPR
from cpr_report_new r
where r.payment_period between date'2024-01-01' and date'2025-12-31'
  and exists (
      select 1
      from secur_cohort s
      where s.con_id = r.con_id
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
   2) Историческая CPR-динамика по "когорте" НЕсекьюритизированных con_id
      (все con_id, которых нет в cpr_exclusions по SECURITIZATION),
      по каждому payment_period
   ============================================================ */

with secur_cohort as (
    select distinct e.con_id
    from cpr_exclusions e
    where e.excl_reason = 'SECURITIZATION'
    -- если нужно строго под сделку 19.12.2025, раскомментируй:
    -- and e.excl_date = date'2025-12-19'
)
select
    r.payment_period as dt_rep,
    case when r.agg_prod_name is null then 'Ипотека Прочее' else r.agg_prod_name end as prod_name,
    sum(r.od_after_plan) as od_after_plan,
    sum(r.od) as od,
    sum(r.premat_payment) as premat,
    case when sum(r.od_after_plan) <= 0 then 0
         else round(100 * (1 - power(1 - sum(r.premat_payment)/sum(r.od_after_plan), 12)), 2)
    end as CPR
from cpr_report_new r
where r.payment_period between date'2024-01-01' and date'2025-12-31'
  and not exists (
      select 1
      from secur_cohort s
      where s.con_id = r.con_id
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
