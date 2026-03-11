Да, суть задачи понимаю:
	•	Шаг 1: перегрузить параметры в s_curve_params_gos (полностью заменить записи по period=0..9 на твой новый набор B0..B6).
	•	Шаг 2: посчитать fact и model CPR по 4 гос-программам и суммарно “Все гос программы” для двух дат payment_period: date'2026-01-31' и date'2026-02-27'.

Ниже даю два блока SQL: (1) замена параметров, (2) расчёт витрины CPR_fact и CPR_model.

⸻


/* ============================================================
   ШАГ 1. Полностью заменяем параметры s_curve_params_gos
   (делай осознанно: это перезапись)
   ============================================================ */

delete from s_curve_params_gos;

insert into s_curve_params_gos (period, b0, b1, b2, b3, b4, b5, b6) values
(0, 0.084018, 0.000000, -2.002365, 2.196743, 0.040095, 1.984944, 0.334097);

insert into s_curve_params_gos (period, b0, b1, b2, b3, b4, b5, b6) values
(1, 0.083262, 0.000000, -2.000553, 2.198757, 0.040369, 1.987474, 0.315231);

insert into s_curve_params_gos (period, b0, b1, b2, b3, b4, b5, b6) values
(2, 0.074119, 0.000000, -2.001046, 2.198592, 0.039298, 1.998372, 0.208560);

insert into s_curve_params_gos (period, b0, b1, b2, b3, b4, b5, b6) values
(3, 0.188848, 0.065923, -1.456688, 1.521187, 0.035984, 0.996996, 0.180874);

insert into s_curve_params_gos (period, b0, b1, b2, b3, b4, b5, b6) values
(4, 0.186330, 0.064299, -1.456688, 1.521179, 0.036002, 0.997004, 0.180528);

insert into s_curve_params_gos (period, b0, b1, b2, b3, b4, b5, b6) values
(5, 0.179954, 0.061224, -1.456696, 1.521166, 0.035728, 0.997262, 0.173819);

insert into s_curve_params_gos (period, b0, b1, b2, b3, b4, b5, b6) values
(6, 0.177209, 0.059366, -1.456699, 1.521160, 0.035837, 0.997279, 0.173809);

insert into s_curve_params_gos (period, b0, b1, b2, b3, b4, b5, b6) values
(7, 0.172046, 0.055895, -1.456706, 1.521103, 0.036016, 0.996622, 0.173609);

insert into s_curve_params_gos (period, b0, b1, b2, b3, b4, b5, b6) values
(8, 0.157777, 0.060220, -1.456751, 1.521023, 0.023243, 0.995356, 0.119569);

insert into s_curve_params_gos (period, b0, b1, b2, b3, b4, b5, b6) values
(9, 0.153494, 0.057322, -1.456724, 1.520814, 0.023412, 0.991638, 0.119489);

commit;


⸻


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

Если у тебя payment_period=2026-02-27 появляется как “не конец месяца”, то всё равно отработает — фильтр стоит ровно на эти две даты, как ты и просил.
