/* ============================================================
   ТОП-5 сделок (con_id) по каждой программе, которые дают
   наибольший вклад в CPR (через вклад в SMM) за декабрь 2025.

   CPR в агрегате считается так же, как в твоём отчёте:
     SMM = sum(premat_payment) / sum(od_after_plan)
     CPR = 100 * (1 - (1 - SMM)^12)

   Вклад сделки i в SMM по программе:
     delta_smm_i = SMM_total - SMM_without_i
                = (P/T) - ((P - p_i)/(T - t_i)),
     где P=sum(premat), T=sum(od_after_plan), p_i=premat_i, t_i=od_after_plan_i

   Дальше ранжируем по delta_smm_i (убывание) и берём топ-5 в каждой программе.
   Можно добавить фильтр на НЕсекьюритизированные, если нужно (см. ниже).
   ============================================================ */

with base as (
    select
        r.con_id,
        r.payment_period,
        case when r.agg_prod_name is null then 'Ипотека Прочее' else r.agg_prod_name end as prod_name,
        r.od_after_plan,
        r.premat_payment
    from cpr_report_new r
    where r.payment_period = date'2025-12-31'
      -- если нужно исключить секьюритизированные con_id из анализа топов, раскомментируй:
      -- and not exists (
      --     select 1
      --     from cpr_exclusions e
      --     where e.con_id = r.con_id
      --       and e.excl_reason = 'SECURITIZATION'
      -- )
),

agg as (
    select
        prod_name,
        sum(od_after_plan) as T,
        sum(premat_payment) as P
    from base
    group by prod_name
),

scored as (
    select
        b.prod_name,
        b.con_id,
        b.od_after_plan as t_i,
        b.premat_payment as p_i,
        a.T,
        a.P,
        /* SMM_total */
        case when a.T <= 0 then 0 else (a.P / a.T) end as smm_total,
        /* SMM_without_i */
        case
            when (a.T - b.od_after_plan) <= 0 then null
            else ( (a.P - b.premat_payment) / (a.T - b.od_after_plan) )
        end as smm_wo_i,
        /* вклад в SMM */
        case
            when a.T <= 0 then 0
            when (a.T - b.od_after_plan) <= 0 then null
            else (a.P / a.T) - ( (a.P - b.premat_payment) / (a.T - b.od_after_plan) )
        end as delta_smm
    from base b
    join agg a
      on a.prod_name = b.prod_name
),

ranked as (
    select
        s.*,
        row_number() over (
            partition by s.prod_name
            order by s.delta_smm desc nulls last, s.p_i desc, s.t_i asc, s.con_id
        ) as rn
    from scored s
)

select
    prod_name,
    con_id,
    round(t_i, 2) as od_after_plan,
    round(p_i, 2) as premat_payment,
    round(100 * smm_total, 6) as smm_total_pct,
    round(100 * delta_smm, 6) as delta_smm_pct,
    /* При желании — "примерный" вклад в CPR как разница CPR_total - CPR_without_i */
    case
        when smm_wo_i is null then null
        else round(
            100 * ( (1 - power(1 - smm_wo_i, 12)) - (1 - power(1 - smm_total, 12)) ) * (-1),
            6
        )
    end as delta_cpr_pp
from ranked
where rn <= 5
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
    prod_name,
    rn;
