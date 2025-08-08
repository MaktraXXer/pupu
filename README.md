/* ── 5b) BASE-DAY HALF-SHIFT: корректный перенос половинок и добавление победителю ─ */
DROP TABLE IF EXISTS #daily_base_adj;

WITH base_pool AS (   -- все строки клиента на base-day
    SELECT
        dp.*,
        is_base = CASE WHEN dp.dt_rep = c.base_day THEN 1 ELSE 0 END
    FROM   #daily_pre dp
    JOIN   #base_dates bd ON bd.cli_id=dp.cli_id AND bd.dt_rep=dp.dt_rep
    LEFT  JOIN #cycles c  ON c.con_id=dp.con_id AND c.cycle_no=dp.cycle_no
),
halves AS (           -- списываем половины С ОКРУГЛЕНИЕМ по каждой базовой строке
    SELECT
        cli_id, dt_rep, con_id, TSEGMENTNAME, rate_con,
        out_half = CASE WHEN is_base=1 THEN ROUND(out_rub*0.5,2) ELSE CAST(0.00 AS decimal(20,2)) END,
        is_base
    FROM base_pool
),
shift AS (            -- ровно та сумма, что сняли (сумма округлённых половин)
    SELECT cli_id, dt_rep, SUM(out_half) AS shift_amt
    FROM halves
    GROUP BY cli_id, dt_rep
),
ranked AS (           -- выбираем победителя по ставке
    SELECT bp.*,
           ROW_NUMBER() OVER (PARTITION BY bp.cli_id, bp.dt_rep
                              ORDER BY bp.rate_con DESC, bp.TSEGMENTNAME, bp.con_id) AS rn
    FROM   base_pool bp
)
SELECT
    r.con_id,
    r.cli_id,
    r.TSEGMENTNAME,
    out_rub =
        CASE
          WHEN r.rn=1 AND r.is_base=1 THEN ROUND(r.out_rub*0.5,2) + s.shift_amt  -- победитель-базовый: половина + перелив
          WHEN r.rn=1 AND r.is_base=0 THEN r.out_rub + s.shift_amt               -- победитель-небазовый: исходный + перелив
          WHEN r.is_base=1                  THEN ROUND(r.out_rub*0.5,2)          -- прочие базовые: оставляем половину
          ELSE                                   r.out_rub                        -- остальным ничего не меняем
        END,
    r.dt_rep,
    r.rate_con
INTO   #daily_base_adj
FROM   ranked r
JOIN   shift  s ON s.cli_id=r.cli_id AND s.dt_rep=r.dt_rep;
