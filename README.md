/* ============================================================
   ТОП-5 сделок (con_id) по каждой программе за декабрь 2025,
   которые дают наибольший вклад в CPR (в терминах ΔCPR),
   + выводим дополнительные поля из cpr_report_new:
     con_rate, age, term, refin_rate, stimul
   ============================================================ */

with base as (
    select
        r.con_id,
        case when r.agg_prod_name is null then 'Ипотека Прочее' else r.agg_prod_name end as prod_name,
        r.od_after_plan,
        r.premat_payment,
        r.con_rate,
        r.age,
        r.term,
        r.refin_rate,
        r.stimul
    from cpr_report_new r
    where r.payment_period = date'2025-12-31'
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
        b.con_rate,
        b.age,
        b.term,
        b.refin_rate,
        b.stimul,
        a.T,
        a.P,

        /* CPR договора (если od_after_plan=0 => 0) */
        case
            when b.od_after_plan <= 0 then 0
            else 100 * (1 - power(1 - (b.premat_payment / b.od_after_plan), 12))
        end as cpr_con,

        /* CPR_total по программе */
        case
            when a.T <= 0 then 0
            else 100 * (1 - power(1 - (a.P / a.T), 12))
        end as cpr_total,

        /* CPR_without_i по программе */
        case
            when (a.T - b.od_after_plan) <= 0 then null
            else 100 * (1 - power(1 - ((a.P - b.premat_payment) / (a.T - b.od_after_plan)), 12))
        end as cpr_wo_i,

        /* delta CPR = CPR_total - CPR_without_i */
        case
            when (a.T - b.od_after_plan) <= 0 then null
            when a.T <= 0 then 0
            else
                (100 * (1 - power(1 - (a.P / a.T), 12)))
                -
                (100 * (1 - power(1 - ((a.P - b.premat_payment) / (a.T - b.od_after_plan)), 12)))
        end as delta_cpr

    from base b
    join agg a
      on a.prod_name = b.prod_name
),

ranked as (
    select
        s.*,
        row_number() over (
            partition by s.prod_name
            order by s.delta_cpr desc nulls last, s.p_i desc, s.t_i asc, s.con_id
        ) as rn
    from scored s
)

select
    prod_name,
    con_id,
    round(t_i, 2) as od_after_plan,
    round(p_i, 2) as premat_payment,
    round(con_rate, 6) as con_rate,
    round(age, 2) as age,
    round(term, 2) as term,
    round(refin_rate, 6) as refin_rate,
    round(stimul, 1) as stimul,
    round(cpr_con, 6) as cpr_con,
    round(cpr_total, 6) as cpr_total,
    round(cpr_wo_i, 6) as cpr_without_i,
    round(delta_cpr, 6) as delta_cpr
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
