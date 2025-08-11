USE [ALM_TEST];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2025-01-01';
DECLARE @DateTo   date = CAST(GETDATE() AS date);

------------------------------------------------------------
-- 1) Снимок из баланса → #src
------------------------------------------------------------
IF OBJECT_ID('tempdb..#src') IS NOT NULL DROP TABLE #src;

SELECT
    t.dt_rep,
    t.dt_open,
    t.con_id,
    CAST(t.out_rub  AS decimal(20,2)) AS out_rub,
    CAST(t.rate_con AS decimal(9,6))  AS rate_balance,
    t.rate_con_src,
    CASE WHEN t.dt_open = t.dt_rep THEN DATEADD(day,1,t.dt_rep) ELSE t.dt_rep END AS eff_dt
INTO #src
FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
  AND t.section_name = N'Накопительный счёт'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.prod_id      = '3103'
  AND t.out_rub IS NOT NULL;

CREATE CLUSTERED INDEX IX_src ON #src(con_id, dt_rep);

------------------------------------------------------------
-- 2) Список нужных договоров → #con
------------------------------------------------------------
IF OBJECT_ID('tempdb..#con') IS NOT NULL DROP TABLE #con;
SELECT DISTINCT con_id INTO #con FROM #src;
CREATE UNIQUE CLUSTERED INDEX IX_con ON #con(con_id);

------------------------------------------------------------
-- 3) LIQUIDITY только по этим договорам → #dcr_small
------------------------------------------------------------
IF OBJECT_ID('tempdb..#dcr_small') IS NOT NULL DROP TABLE #dcr_small;
SELECT r.con_id, r.dt_from, r.dt_to, r.rate
INTO #dcr_small
FROM LIQUIDITY.liq.DepositContract_Rate r WITH (NOLOCK)
JOIN #con c ON c.con_id = r.con_id;

CREATE NONCLUSTERED INDEX IX_dcr_small_seek
ON #dcr_small(con_id, dt_from DESC) INCLUDE (dt_to, rate);

------------------------------------------------------------
-- 4) Договорная ставка (cover-first / fallback) → #bal
------------------------------------------------------------
IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;

SELECT
    s.dt_rep, s.dt_open, s.con_id, s.out_rub, s.rate_balance, s.rate_con_src,
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
      ISNULL(d.dt_to, '9999-12-31')
) AS pick;

CREATE CLUSTERED INDEX IX_bal ON #bal(con_id, dt_rep);

------------------------------------------------------------
-- 5) rate_use по твоим правилам → #rc0
------------------------------------------------------------
IF OBJECT_ID('tempdb..#rc0') IS NOT NULL DROP TABLE #rc0;

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
    bp.*,
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
    ) AS rate_use
INTO #rc0
FROM bal_pos bp;

CREATE NONCLUSTERED INDEX IX_rc0_dt ON #rc0(dt_rep) INCLUDE(out_rub, rate_use);

------------------------------------------------------------
-- 6) Периоды ключевой ставки → #key_p
------------------------------------------------------------
IF OBJECT_ID('tempdb..#key_p') IS NOT NULL DROP TABLE #key_p;

;WITH k AS (
    SELECT
        k.dt_rep                                AS dt_from,
        LEAD(k.dt_rep) OVER (ORDER BY k.dt_rep) AS dt_next,
        CAST(k.KEY_RATE AS decimal(18,6))       AS key_rate
    FROM ALM_TEST.price.key_rate_fact k WITH (NOLOCK)
)
SELECT *
INTO #key_p
FROM k
WHERE (dt_next IS NULL OR dt_next > @DateFrom) AND dt_from <= @DateTo;

CREATE NONCLUSTERED INDEX IX_key_p ON #key_p(dt_from, dt_next);

------------------------------------------------------------
-- 7) Справочник бакетов → #bucket_map
------------------------------------------------------------
IF OBJECT_ID('tempdb..#bucket_map') IS NOT NULL DROP TABLE #bucket_map;

CREATE TABLE #bucket_map (
    bucket_idx int,
    from_spread decimal(9,6),
    to_spread decimal(9,6),
    bucket_label nvarchar(50)
);

INSERT INTO #bucket_map(bucket_idx, from_spread, to_spread, bucket_label) VALUES
 (0, NULL      , 0.000000 , N'< 0'),
 (1, 0.000000  , 0.009000 , N'0 – 0.9 п.п.'),
 (2, 0.009000  , 0.012000 , N'0.9 – 1.2 п.п.'),
 (3, 0.012000  , 0.020000 , N'1.2 – 2.0 п.п.'),
 (4, 0.020000  , 0.050000 , N'2.0 – 5.0 п.п.'),
 (5, 0.050000  , NULL     , N'5%+'),
 (9, NULL      , NULL     , N'Н/Д');

------------------------------------------------------------
-- 8) Присоединяем ключевую, считаем spread и бакет → #rc
------------------------------------------------------------
IF OBJECT_ID('tempdb..#rc') IS NOT NULL DROP TABLE #rc;

SELECT
    r.*,
    k.key_rate,
    CASE
      WHEN r.rate_use IS NULL OR k.key_rate IS NULL THEN NULL
      ELSE k.key_rate - r.rate_use
    END AS spread,
    bm.bucket_idx,
    bm.bucket_label
INTO #rc
FROM #rc0 r
JOIN #key_p k
  ON r.dt_rep >= k.dt_from
 AND (k.dt_next IS NULL OR r.dt_rep < k.dt_next)
OUTER APPLY (
    SELECT TOP (1) b.bucket_idx, b.bucket_label
    FROM #bucket_map b
    WHERE
        (r.rate_use IS NULL OR k.key_rate IS NULL AND b.bucket_idx = 9)
     OR (r.rate_use IS NOT NULL AND k.key_rate IS NOT NULL
         AND (
              (b.from_spread IS NULL OR (k.key_rate - r.rate_use) >= b.from_spread)
          AND (b.to_spread   IS NULL OR (k.key_rate - r.rate_use) <  b.to_spread)
         )
        )
    ORDER BY b.bucket_idx
) bm;

CREATE NONCLUSTERED INDEX IX_rc_dt_bucket ON #rc(dt_rep, bucket_idx) INCLUDE(out_rub, rate_use, spread);

------------------------------------------------------------
-- 9) Панель 1: весь портфель
------------------------------------------------------------
IF OBJECT_ID('tempdb..#panel_all') IS NOT NULL DROP TABLE #panel_all;

SELECT
    dt_rep,
    SUM(out_rub) AS out_rub_total,
    SUM(CASE WHEN rate_use IS NOT NULL THEN out_rub END) AS out_rub_with_rate,
    CAST(
      SUM(CASE WHEN rate_use IS NOT NULL THEN rate_use * out_rub END)
      / NULLIF(SUM(CASE WHEN rate_use IS NOT NULL THEN out_rub END),0)
      AS decimal(9,6)
    ) AS rate_con
INTO #panel_all
FROM #rc
GROUP BY dt_rep;

------------------------------------------------------------
-- 10) Панель 2: по бакетам
------------------------------------------------------------
IF OBJECT_ID('tempdb..#panel_bucket') IS NOT NULL DROP TABLE #panel_bucket;

SELECT
    dt_rep,
    bucket_idx,
    bucket_label,
    SUM(out_rub) AS out_rub_total,
    SUM(CASE WHEN rate_use IS NOT NULL THEN out_rub END) AS out_rub_with_rate,
    CAST(
      SUM(CASE WHEN rate_use IS NOT NULL THEN rate_use * out_rub END)
      / NULLIF(SUM(CASE WHEN rate_use IS NOT NULL THEN out_rub END),0)
      AS decimal(9,6)
    ) AS rate_con,
    CAST(AVG(spread) AS decimal(9,6)) AS avg_spread
INTO #panel_bucket
FROM #rc
GROUP BY dt_rep, bucket_idx, bucket_label;

------------------------------------------------------------
-- 11) Результаты
------------------------------------------------------------
SELECT * FROM #panel_all ORDER BY dt_rep;
SELECT * FROM #panel_bucket ORDER BY dt_rep, bucket_idx;
