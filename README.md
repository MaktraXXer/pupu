USE [ALM_TEST];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2025-07-01';
DECLARE @DateTo   date = '2025-07-31';

------------------------------------------------------------
-- 1) Баланс (prod_id=654) → #src
--    Берём +0 дней к началу; для проверок "последний день месяца" предыдущее значение внутри месяца есть.
------------------------------------------------------------
IF OBJECT_ID('tempdb..#src') IS NOT NULL DROP TABLE #src;

SELECT
    t.dt_rep,
    t.dt_open,
    t.con_id,
    CAST(t.out_rub  AS decimal(20,2)) AS out_rub,
    CAST(t.rate_con AS decimal(9,6))  AS rate_balance,
    -- месячные ключи
    DATEFROMPARTS(YEAR(t.dt_rep),  MONTH(t.dt_rep),  1) AS rep_month,
    DATEFROMPARTS(YEAR(t.dt_open), MONTH(t.dt_open), 1) AS open_month,
    CASE WHEN t.dt_open = t.dt_rep THEN DATEADD(day,1,t.dt_rep) ELSE t.dt_rep END AS eff_dt
INTO #src
FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
  AND t.section_name = N'Накопительный счёт'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.prod_id      = 654
  AND t.out_rub IS NOT NULL;

CREATE CLUSTERED INDEX IX_src ON #src(con_id, dt_rep);

------------------------------------------------------------
-- 2) Ужимаем LIQUIDITY до нужных договоров → #dcr_small
------------------------------------------------------------
IF OBJECT_ID('tempdb..#con') IS NOT NULL DROP TABLE #con;
SELECT DISTINCT con_id INTO #con FROM #src;
CREATE UNIQUE CLUSTERED INDEX IX_con ON #con(con_id);

IF OBJECT_ID('tempdb..#dcr_small') IS NOT NULL DROP TABLE #dcr_small;
SELECT r.con_id, r.dt_from, r.dt_to, r.rate
INTO #dcr_small
FROM LIQUIDITY.liq.DepositContract_Rate r WITH (NOLOCK)
JOIN #con c ON c.con_id = r.con_id;

CREATE NONCLUSTERED INDEX IX_dcr_small_seek
ON #dcr_small(con_id, dt_from DESC) INCLUDE (dt_to, rate);

------------------------------------------------------------
-- 3) Договорная ставка: покрывающая eff_dt, иначе последняя до eff_dt → #bal
------------------------------------------------------------
IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;

SELECT
    s.dt_rep, s.dt_open, s.con_id, s.out_rub, s.rate_balance,
    s.rep_month, s.open_month,
    DATEDIFF(MONTH, s.open_month, s.rep_month) AS age_months,  -- 0/1/2/3+
    pick.rate AS rate_liq
INTO #bal
FROM #src s
OUTER APPLY (
    SELECT TOP (1) d.rate
    FROM #dcr_small d
    WHERE d.con_id = s.con_id
      AND d.dt_from <= s.eff_dt
    ORDER BY
      CASE WHEN (d.dt_to IS NULL OR d.dt_to >= s.eff_dt) THEN 0 ELSE 1 END,
      d.dt_from DESC,
      ISNULL(d.dt_to,'9999-12-31')
) AS pick;

CREATE CLUSTERED INDEX IX_bal ON #bal(con_id, dt_rep);

------------------------------------------------------------
-- 4) Итоговые флаги + rate_use (без «ультра») → #rc
------------------------------------------------------------
IF OBJECT_ID('tempdb..#rc') IS NOT NULL DROP TABLE #rc;

SELECT
    b.dt_rep,
    b.dt_open,
    b.con_id,
    b.out_rub,
    CAST(COALESCE(b.rate_liq, b.rate_balance) AS decimal(9,6)) AS rate_use,
    b.rep_month,
    b.open_month,
    b.age_months,
    CASE
      WHEN b.age_months = 0 THEN N'M0'
      WHEN b.age_months = 1 THEN N'M-1'
      WHEN b.age_months = 2 THEN N'M-2'
      WHEN b.age_months >= 3 THEN N'older'
      ELSE N'unknown'
    END AS age_bucket,
    CASE
      WHEN b.dt_open <  '2025-07-01' AND b.age_months <= 2 THEN N'promo'     -- 3 мес: 0,1,2
      WHEN b.dt_open >= '2025-07-01' AND b.age_months <= 1 THEN N'promo'     -- 2 мес: 0,1
      ELSE N'non_promo'
    END AS promo_flag
INTO #rc
FROM #bal b;

CREATE NONCLUSTERED INDEX IX_rc_dt ON #rc(dt_rep, age_bucket, promo_flag) INCLUDE(out_rub, rate_use, rep_month, con_id);

------------------------------------------------------------
-- 5) Месячный минимум по счёту → #rc_mmin
--    Берём MIN(out_rub) по (con_id, rep_month); его используем как вес.
------------------------------------------------------------
IF OBJECT_ID('tempdb..#rc_mmin') IS NOT NULL DROP TABLE #rc_mmin;

SELECT
    r.*,
    MIN(r.out_rub) OVER (PARTITION BY r.con_id, r.rep_month) AS out_rub_mmin
INTO #rc_mmin
FROM #rc r;

CREATE NONCLUSTERED INDEX IX_rc_mmin ON #rc_mmin(dt_rep, age_bucket, promo_flag) INCLUDE(out_rub_mmin, rate_use);

------------------------------------------------------------
-- A) ПАНЕЛЬ «день × возраст × промо», где объём = месячный минимум по счёту
------------------------------------------------------------
SELECT
    r.dt_rep,
    r.age_bucket,
    r.promo_flag,
    SUM(r.out_rub_mmin) AS out_rub_total_mmin,
    SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub_mmin END) AS out_rub_with_rate_mmin,
    CAST(
      SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.rate_use * r.out_rub_mmin END)
      / NULLIF(SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub_mmin END),0)
      AS decimal(9,6)
    ) AS rate_con_mmin
FROM #rc_mmin r
GROUP BY r.dt_rep, r.age_bucket, r.promo_flag
ORDER BY r.dt_rep, r.age_bucket, r.promo_flag;

------------------------------------------------------------
-- B) Сверка: сколько счетов имели Δостатка > 40% в ПОСЛЕДНИЙ день месяца
--    Δ считаем day-over-day: |out_t - out_{t-1}| / out_{t-1} > 0.4
------------------------------------------------------------
WITH chg AS (
    SELECT
        s.con_id,
        s.dt_rep,
        s.out_rub,
        LAG(s.out_rub) OVER (PARTITION BY s.con_id ORDER BY s.dt_rep) AS prev_out
    FROM #src s
),
month_end AS (
    SELECT *
    FROM chg
    WHERE dt_rep = EOMONTH(dt_rep)
      AND dt_rep BETWEEN @DateFrom AND @DateTo
)
SELECT
    dt_rep AS month_end_date,
    COUNT(CASE WHEN prev_out IS NOT NULL AND prev_out > 0 THEN 1 END)              AS accounts_with_prev,
    SUM(CASE WHEN prev_out IS NOT NULL AND prev_out > 0
              AND ABS(out_rub - prev_out)/prev_out > 0.40 THEN 1 ELSE 0 END)      AS accounts_change_gt_40pct,
    CAST(
      SUM(CASE WHEN prev_out IS NOT NULL AND prev_out > 0
                AND ABS(out_rub - prev_out)/prev_out > 0.40 THEN 1 ELSE 0 END) * 1.0
      / NULLIF(COUNT(CASE WHEN prev_out IS NOT NULL AND prev_out > 0 THEN 1 END),0)
      AS decimal(9,4)
    ) AS share_gt_40pct
FROM month_end
GROUP BY dt_rep
ORDER BY month_end_date;
