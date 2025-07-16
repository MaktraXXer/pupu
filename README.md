/* ===== параметры периода ========================================== */
DECLARE @DateFrom date = '2025‑05‑01',
        @DateTo   date = '2025‑07‑01';

/* ===== 1. Срочные (все валюты) ==================================== */
WITH term AS (
    SELECT  t.dt_rep,
            t.out_rub,
            /* эквивалентная рублёвая ставка */
            t.rate_con * t.out_cur / NULLIF(t.out_rub,0) AS rate_rub
    FROM    alm.ALM.vw_balance_rest_all t
    WHERE   t.dt_rep BETWEEN @DateFrom AND @DateTo
      AND   t.section_name =  N'Срочные'
      AND   t.block_name   =  N'Привлечение ФЛ'
      AND   t.od_flag      = 1            -- действующие
),

/* ===== 2. НС + ДВС: та же логика, что в usp_fill_balance_metrics_savings */
bal0 AS (
    SELECT  t.dt_rep, t.dt_open, t.con_id,
            t.out_cur,
            t.out_rub,
            t.rate_con          AS rate_balance,
            t.rate_con_src,
            r.rate              AS rate_liq
    FROM    alm.ALM.vw_balance_rest_all t
    LEFT    JOIN LIQUIDITY.liq.DepositContract_Rate r
             ON  r.con_id = t.con_id
             AND CASE WHEN t.dt_open = t.dt_rep
                      THEN DATEADD(day,1,t.dt_rep)     -- «нулевой» день ULTRA
                      ELSE t.dt_rep
                 END BETWEEN r.dt_from AND r.dt_to
    WHERE   t.dt_rep BETWEEN @DateFrom AND @DateTo
      AND   t.section_name IN (N'Накопительный счёт', N'До востребования')
      AND   t.block_name   =  N'Привлечение ФЛ'
      AND   t.od_flag      = 1
),
bal_pos AS (
    SELECT *, MIN(CASE WHEN rate_balance>0
                        AND rate_con_src=N'счет ультра,вручную'
                       THEN rate_balance END)
                 OVER (PARTITION BY con_id
                       ORDER BY dt_rep
                       ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
    FROM   bal0
),
rate_calc AS (
    SELECT *, CASE
        WHEN rate_liq IS NULL
             THEN CASE WHEN rate_balance<0
                       THEN COALESCE(rate_pos, rate_balance)
                       ELSE rate_balance END
        WHEN rate_liq<0  AND rate_balance>0  THEN rate_balance
        WHEN rate_liq<0  AND rate_balance<0  THEN COALESCE(rate_pos, rate_balance)
        WHEN rate_liq>=0 AND rate_balance>=0 THEN rate_liq
        WHEN rate_liq>0  AND rate_balance<0  THEN rate_liq
        ELSE rate_liq
    END AS rate_use
    FROM bal_pos
),
sav_dem AS (
    SELECT dt_rep,
           out_rub,
           /* переводим ставку в рубли */ 
           rate_use * out_cur / NULLIF(out_rub,0) AS rate_rub
    FROM   rate_calc
),

/* ===== 3. Собираем всё и агрегируем ================================ */
all_rows AS (
    SELECT dt_rep, out_rub, rate_rub FROM term
    UNION ALL
    SELECT dt_rep, out_rub, rate_rub FROM sav_dem
),
aggr AS (
    SELECT dt_rep,
           SUM(out_rub)                             AS out_rub_total,
           SUM(rate_rub*out_rub)/NULLIF(SUM(out_rub),0) AS rate_wavg
    FROM   all_rows
    GROUP BY dt_rep
)
SELECT dt_rep,
       out_rub_total,
       rate_wavg
FROM   aggr
ORDER  BY dt_rep;

