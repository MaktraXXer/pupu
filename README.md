select
    c.con_id,
    /* 'Y' если есть актуальный WITHDRAWAL, иначе NULL */
    max(case when a.con_attr_code = 'WITHDRAWAL_OPTION' then 'Y' end) as attr_WITHDRAWAL,
    /* 'Y' если есть актуальный REPLENISH, иначе NULL */
    max(case when a.con_attr_code = 'REPLENISH_OPTION'  then 'Y' end) as attr_REPLENISH
from dds.contract c
join dds.con_attr a
  on a.con_id = c.con_id
 and a.con_attr_val  = 'Y'
 and a.con_attr_code in ('WITHDRAWAL_OPTION', 'REPLENISH_OPTION')
 /* интервал актуальности атрибута на момент открытия договора — через BETWEEN без OR */
 and c.dt_open between a.dt_from and nvl(a.dt_to, date '9999-12-31')
where c.con_type = 'DEPOSIT'
  /* фильтр по дате открытия договора — тоже через BETWEEN */
  and c.dt_open between date '2025-09-01' and date '9999-12-31'
group by c.con_id
/* «вариант A»: есть хотя бы одна из двух опций (OR), без явного OR в SQL */
having
    max(case when a.con_attr_code = 'WITHDRAWAL_OPTION' then 1 else 0 end)
  + max(case when a.con_attr_code = 'REPLENISH_OPTION'  then 1 else 0 end) >= 1;
