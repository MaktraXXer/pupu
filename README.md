select
    c.con_id,
    /* 1/0-флаги по актуальным на дату открытия атрибутам */
    max(case when a.con_attr_code = 'WITHDRAWAL_OPTION' then 1 else 0 end) as attr_WITHDRAWAL,
    max(case when a.con_attr_code = 'REPLENISH_OPTION' then 1 else 0 end)  as attr_REPLENISH
from dds.contract c
join dds.con_attr a
  on a.con_id = c.con_id
 /* актуальность атрибута на дату открытия вклада */
 and a.dt_from <= c.dt_open
 and (a.dt_to is null or c.dt_open <= a.dt_to)    -- надёжнее half-open, чем BETWEEN при null
 /* интересуют только нужные коды и только положительные значения */
 and a.con_attr_val = 'Y'
 and a.con_attr_code in ('WITHDRAWAL_OPTION','REPLENISH_OPTION')
where c.con_type = 'DEPOSIT'
  and c.dt_open >= date '2025-09-01'
group by c.con_id
having
    /* «вариант А»: есть хотя бы одна из двух опций */
    max(case when a.con_attr_code = 'WITHDRAWAL_OPTION' then 1 else 0 end) = 1
 or max(case when a.con_attr_code = 'REPLENISH_OPTION' then 1 else 0 end)  = 1;
