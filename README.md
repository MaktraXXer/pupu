/* 
  Oracle / PL/SQL Developer (обычный SQL Statement)

  Меняешь ТОЛЬКО dt_rep в CTE params.
  Флаги:
    flag_curr_month  = открытия в календарном месяце dt_rep
    flag_prev_month  = открытия в календарном месяце (dt_rep - 1 месяц)
  Срез ставок/атрибутов/остатков — на дату dt_rep.
*/

WITH
params AS (
    SELECT
        DATE '2025-11-24' AS dt_rep,
        654 AS prod_id,
        810 AS cur
    FROM dual
),
base AS (
    SELECT
        c.cli_id,
        c.prod_id,
        c.con_id,
        c.dt_open,
        s.out_rub,
        t.con_rate AS rate_balance,
        j.con_attr_val
    FROM dds.contract c
    CROSS JOIN params p
    LEFT JOIN dds.con_rate t
           ON c.con_id = t.con_id
          AND p.dt_rep BETWEEN t.dt_from AND t.dt_to
    LEFT JOIN dm_common.cdm_con_rest s
           ON c.con_id = s.con_id
          AND s.dt_rep = p.dt_rep
    LEFT JOIN dds.con_attr j
           ON c.con_id = j.con_id
          AND j.con_attr_code = 'CLI_BIZ'
          AND p.dt_rep BETWEEN j.dt_from AND j.dt_to
    WHERE c.prod_id = p.prod_id
      AND c.cur     = p.cur
      AND c.dt_close_fact > p.dt_rep
),
flag_prev_month AS (
    SELECT DISTINCT
        c.cli_id,
        1 AS has_prev_month
    FROM dds.contract c
    CROSS JOIN params p
    WHERE c.prod_id = p.prod_id
      AND c.cur     = p.cur
      AND c.dt_open >= TRUNC(ADD_MONTHS(p.dt_rep, -1), 'MM')
      AND c.dt_open <  TRUNC(p.dt_rep, 'MM')
      AND c.dt_close_fact > p.dt_rep
),
flag_curr_month AS (
    SELECT DISTINCT
        c.cli_id,
        1 AS has_curr_month
    FROM dds.contract c
    CROSS JOIN params p
    WHERE c.prod_id = p.prod_id
      AND c.cur     = p.cur
      AND c.dt_open >= TRUNC(p.dt_rep, 'MM')
      AND c.dt_open <  ADD_MONTHS(TRUNC(p.dt_rep, 'MM'), 1)
      AND c.dt_close_fact > p.dt_rep
)
SELECT
    b.cli_id,
    b.con_id,
    b.dt_open,
    b.out_rub,
    b.rate_balance,
    b.con_attr_val,
    NVL(pm.has_prev_month, 0) AS flag_prev_month,
    NVL(cm.has_curr_month, 0) AS flag_curr_month
FROM base b
LEFT JOIN flag_prev_month pm
       ON b.cli_id = pm.cli_id
LEFT JOIN flag_curr_month cm
       ON b.cli_id = cm.cli_id
ORDER BY b.cli_id, b.dt_open;
