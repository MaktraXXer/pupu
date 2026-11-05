
WITH params AS (
    SELECT
        DATE '2026-09-29' AS dt_from,
        DATE '2026-11-05' AS dt_to
),
contracts AS (
    SELECT DISTINCT
        c.dt_first_act AS dt,
        r.con_id
    FROM dds.con_rel r
    JOIN dds.contract c
        ON c.con_id = r.con_id_rel
    CROSS JOIN params p
    WHERE r.con_rel_type = 'SECURITIZATION'
      AND c.bal_holder = 'DOM'
      AND c.dt_first_act BETWEEN p.dt_from AND p.dt_to
)
SELECT
    TO_CHAR(c.dt, 'DD') || ' ' ||
    CASE EXTRACT(MONTH FROM c.dt)
        WHEN 1 THEN 'янв'
        WHEN 2 THEN 'фев'
        WHEN 3 THEN 'мар'
        WHEN 4 THEN 'апр'
        WHEN 5 THEN 'май'
        WHEN 6 THEN 'июн'
        WHEN 7 THEN 'июл'
        WHEN 8 THEN 'авг'
        WHEN 9 THEN 'сен'
        WHEN 10 THEN 'окт'
        WHEN 11 THEN 'ноя'
        WHEN 12 THEN 'дек'
    END AS "Дата",
    SUM(s.out_rub) AS "Объем"
FROM contracts c
JOIN dm_common.cdm_con_rest s
    ON s.con_id = c.con_id
   AND s.dt_rep = c.dt - 1
GROUP BY c.dt
ORDER BY c.dt;
