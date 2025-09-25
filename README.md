select
    c.con_id,
    case when exists (
        select 1
        from dds.con_attr a
        where a.con_id = c.con_id
          and a.con_attr_code = 'WITHDRAWAL_OPTION'
          and a.con_attr_val  = 'Y'
          and c.dt_open between a.dt_from and a.dt_to
    ) then 1 else 0 end as attr_WITHDRAWAL,
    case when exists (
        select 1
        from dds.con_attr a
        where a.con_id = c.con_id
          and a.con_attr_code = 'REPLENISH_OPTION'
          and a.con_attr_val  = 'Y'
          and c.dt_open between a.dt_from and a.dt_to
    ) then 1 else 0 end as attr_REPLENISH
from dds.contract c
where c.con_type = 'DEPOSIT'
  and c.dt_open >= DATE '2025-09-01'
  and (
        exists (
            select 1 from dds.con_attr a
            where a.con_id = c.con_id
              and a.con_attr_code = 'WITHDRAWAL_OPTION'
              and a.con_attr_val  = 'Y'
              and c.dt_open between a.dt_from and a.dt_to
        )
     or exists (
            select 1 from dds.con_attr a
            where a.con_id = c.con_id
              and a.con_attr_code = 'REPLENISH_OPTION'
              and a.con_attr_val  = 'Y'
              and c.dt_open between a.dt_from and a.dt_to
        )
  );
