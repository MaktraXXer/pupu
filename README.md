USE [ALM_TEST];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2025-01-01';
DECLARE @DateTo   date = CAST(GETDATE() AS date);

------------------------------------------------------------
-- 1) Баланс → #src  (prod_id=654, «портфель»)
------------------------------------------------------------
IF OBJECT_ID('tempdb..#src') IS NOT NULL DROP TABLE #src;

SELECT
    t.dt_rep,
    t.dt_open,
    t.con_id,
    CAST(t.out_rub  AS decimal(20,2)) AS out_rub,
    CAST(t.rate_con AS decimal(9,6))  AS rate_balance,
    t.rate_con_src,
    -- для "возраста" по месяцам:
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
-- 2) Ужимаем LIQUIDITY → #dcr_small
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
-- 3) Договорная ставка на дату (cover-first / fallback одним APPLY) → #bal
------------------------------------------------------------
IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;

SELECT
    s.dt_rep, s.dt_open, s.con_id, s.out_rub, s.rate_balance, s.rate_con_src,
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
-- 4) rate_use + флаги возраста и ПРОМО → #rc
------------------------------------------------------------
IF OBJECT_ID('tempdb..#rc') IS NOT NULL DROP TABLE #rc;

;WITH bal_pos AS (
    SELECT  b.*,
            MIN(CASE WHEN b.rate_balance > 0
                      AND b.rate_con_src = N'счет ультра,вручную'
                     THEN b.rate_balance END)
                OVER (PARTITION BY b.con_id
                      ORDER BY b.dt_rep
                      ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
    FROM #bal b
)
SELECT
    bp.dt_rep,
    bp.dt_open,
    bp.con_id,
    bp.out_rub,
    bp.rate_balance,
    bp.rate_con_src,
    bp.age_months,
    CASE
      WHEN bp.age_months = 0 THEN N'M0'
      WHEN bp.age_months = 1 THEN N'M-1'
      WHEN bp.age_months = 2 THEN N'M-2'
      WHEN bp.age_months >= 3 THEN N'older'
      ELSE N'unknown'
    END AS age_bucket,
    -- итоговая ставка:
    CAST(
        CASE
          WHEN bp.rate_liq IS NULL THEN
               CASE WHEN bp.rate_balance < 0
                        THEN COALESCE(bp.rate_pos, bp.rate_balance)
                    ELSE bp.rate_balance
               END
          WHEN bp.rate_liq < 0  AND bp.rate_balance > 0 THEN bp.rate_balance
          WHEN bp.rate_liq < 0  AND bp.rate_balance < 0 THEN COALESCE(bp.rate_pos, bp.rate_balance)
          WHEN bp.rate_liq >= 0 AND bp.rate_balance >= 0 THEN bp.rate_liq
          WHEN bp.rate_liq > 0  AND bp.rate_balance  < 0 THEN bp.rate_liq
          ELSE bp.rate_liq
        END AS decimal(9,6)
    ) AS rate_use,
    -- промо-правило: по месяцу открытия
    CASE
      WHEN bp.dt_open <  '2025-07-01' AND bp.age_months <= 2 THEN N'promo'     -- 3 месяца: 0,1,2
      WHEN bp.dt_open >= '2025-07-01' AND bp.age_months <= 1 THEN N'promo'     -- 2 месяца: 0,1
      ELSE N'non_promo'
    END AS promo_flag
INTO #rc
FROM bal_pos bp;

CREATE NONCLUSTERED INDEX IX_rc_dt ON #rc(dt_rep, age_bucket, promo_flag) INCLUDE(out_rub, rate_use);

------------------------------------------------------------
-- 5) ПАНЕЛЬ 1: весь портфель (день → объём, средняя ставка)
------------------------------------------------------------
SELECT
    r.dt_rep,
    SUM(r.out_rub) AS out_rub_total,
    SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub END) AS out_rub_with_rate,
    CAST(
      SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.rate_use * r.out_rub END)
      / NULLIF(SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub END),0)
      AS decimal(9,6)
    ) AS rate_con
FROM #rc r
GROUP BY r.dt_rep
ORDER BY r.dt_rep;

------------------------------------------------------------
-- 6) ПАНЕЛЬ 2: по возрастным флагам (M0 / M-1 / M-2 / older)
------------------------------------------------------------
SELECT
    r.dt_rep,
    r.age_bucket,
    SUM(r.out_rub) AS out_rub_total,
    SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub END) AS out_rub_with_rate,
    CAST(
      SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.rate_use * r.out_rub END)
      / NULLIF(SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub END),0)
      AS decimal(9,6)
    ) AS rate_con
FROM #rc r
GROUP BY r.dt_rep, r.age_bucket
ORDER BY r.dt_rep, r.age_bucket;

------------------------------------------------------------
-- 7) ПАНЕЛЬ 3: по PROMO / NON_PROMO
------------------------------------------------------------
SELECT
    r.dt_rep,
    r.promo_flag,
    SUM(r.out_rub) AS out_rub_total,
    SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub END) AS out_rub_with_rate,
    CAST(
      SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.rate_use * r.out_rub END)
      / NULLIF(SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub END),0)
      AS decimal(9,6)
    ) AS rate_con
FROM #rc r
GROUP BY r.dt_rep, r.promo_flag
ORDER BY r.dt_rep, r.promo_flag;
