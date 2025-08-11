USE [ALM_TEST];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2025-01-01';
DECLARE @DateTo   date = CAST(GETDATE() AS date);

-- Если у вас ставки (и key rate, и договорные) уже в процентах, оставьте @scale_* = 1.
-- Если в долях (0.105), поставьте 100.
DECLARE @scale_key  decimal(6,2) = 1;     -- 1 или 100
DECLARE @scale_liq  decimal(6,2) = 1;     -- 1 или 100

-- Допуск сравнения в процентных пунктах после масштабирования:
DECLARE @eps_pp decimal(9,4) = 0.01;      -- 0.01 п.п.

------------------------------------------------------------
-- 1) Снимок из баланса (ТОЛЬКО нужный диапазон) → #src
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

-- Кластерный индекс помогает всем последующим оконным и APPLY
CREATE CLUSTERED INDEX IX_src ON #src(con_id, dt_rep);

------------------------------------------------------------
-- 2) Предфильтруем LIQUIDITY по набору con_id → #dcr_small
--    Это резко режет объём сканов по большой таблице DCR
------------------------------------------------------------
IF OBJECT_ID('tempdb..#con') IS NOT NULL DROP TABLE #con;
SELECT DISTINCT s.con_id INTO #con FROM #src s;
CREATE UNIQUE CLUSTERED INDEX IX_con ON #con(con_id);

IF OBJECT_ID('tempdb..#dcr_small') IS NOT NULL DROP TABLE #dcr_small;
SELECT r.con_id, r.dt_from, r.dt_to, r.rate
INTO #dcr_small
FROM LIQUIDITY.liq.DepositContract_Rate r WITH (NOLOCK)
JOIN #con c ON c.con_id = r.con_id;
/* Индекс под поиск «последний до даты / покрывающий дату»: */
CREATE NONCLUSTERED INDEX IX_dcr_small_seek
ON #dcr_small(con_id, dt_from DESC)
INCLUDE (dt_to, rate);

------------------------------------------------------------
-- 3) Присоединяем договорную ставку (rate_liq) к #src → #bal
--    Ищем запись, которая покрывает eff_dt; если нет — берём
--    последнюю до eff_dt. Всё через один APPLY, который SEEK-ится.
------------------------------------------------------------
IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;

SELECT
    s.dt_rep, s.dt_open, s.con_id, s.out_rub, s.rate_balance, s.rate_con_src,
    da.rate AS rate_liq
INTO #bal
FROM #src s
OUTER APPLY (
    SELECT TOP (1) d.rate
    FROM #dcr_small d
    WHERE d.con_id = s.con_id
      AND d.dt_from <= s.eff_dt
      AND (d.dt_to IS NULL OR d.dt_to >= s.eff_dt)
    ORDER BY d.dt_from DESC, ISNULL(d.dt_to,'9999-12-31')
) da
OUTER APPLY (
    -- Фоллбэк: если покрывающей нет, берём последнюю до eff_dt
    SELECT TOP (1) d.rate
    FROM #dcr_small d
    WHERE da.rate IS NULL             -- фоллбэк нужен
      AND d.con_id = s.con_id
      AND d.dt_from <= s.eff_dt
    ORDER BY d.dt_from DESC
) df
-- Выбираем покрывающую, иначе фоллбэк
UPDATE #bal SET rate_liq = COALESCE(rate_liq, df.rate)
FROM #bal b
JOIN (
    SELECT s.dt_rep, s.con_id,
           COALESCE(da.rate, df.rate) AS rate_liq
    FROM #src s
    OUTER APPLY (
        SELECT TOP (1) d.rate
        FROM #dcr_small d
        WHERE d.con_id = s.con_id
          AND d.dt_from <= s.eff_dt
          AND (d.dt_to IS NULL OR d.dt_to >= s.eff_dt)
        ORDER BY d.dt_from DESC, ISNULL(d.dt_to,'9999-12-31')
    ) da
    OUTER APPLY (
        SELECT TOP (1) d.rate
        FROM #dcr_small d
        WHERE da.rate IS NULL
          AND d.con_id = s.con_id
          AND d.dt_from <= s.eff_dt
        ORDER BY d.dt_from DESC
    ) df
) u ON u.dt_rep = b.dt_rep AND u.con_id = b.con_id;

-- Индекс под оконку по con_id,dt_rep
CREATE CLUSTERED INDEX IX_bal ON #bal(con_id, dt_rep);

------------------------------------------------------------
-- 4) Рабочая ставка rate_use (как у тебя), с rate_pos
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

CREATE NONCLUSTERED INDEX IX_rc0_dt ON #rc0(dt_rep) INCLUDE(out_rub, rate_use, rate_liq);

------------------------------------------------------------
-- 5) Периоды ключевой ставки (one scan, no per-row lookup) → #key_p
------------------------------------------------------------
IF OBJECT_ID('tempdb..#key_p') IS NOT NULL DROP TABLE #key_p;

;WITH k AS (
    SELECT
        k.dt_rep AS dt_from,
        LEAD(k.dt_rep) OVER (ORDER BY k.dt_rep) AS dt_next,
        CAST(k.KEY_RATE * @scale_key AS decimal(18,6)) AS key_rate_pct
    FROM ALM_TEST.price.key_rate_fact k WITH (NOLOCK)
),
kp AS (
    SELECT
        dt_from,
        ISNULL(DATEADD(day,-1,dt_next), '9999-12-31') AS dt_to,
        key_rate_pct
    FROM k
)
SELECT * INTO #key_p FROM kp
WHERE dt_to >= @DateFrom AND dt_from <= @DateTo;

CREATE NONCLUSTERED INDEX IX_key_p ON #key_p(dt_from, dt_to);

------------------------------------------------------------
-- 6) Присоединяем ключевую по диапазону + строим бакеты → #rc
------------------------------------------------------------
IF OBJECT_ID('tempdb..#rc') IS NOT NULL DROP TABLE #rc;

SELECT
    r.*,
    k.key_rate_pct,
    CAST(r.rate_liq * @scale_liq AS decimal(18,6)) AS rate_liq_pct,
    CASE
      WHEN r.rate_liq IS NULL OR k.key_rate_pct IS NULL THEN NULL
      WHEN ABS( (r.rate_liq * @scale_liq) - (k.key_rate_pct - 2.0) ) <= @eps_pp THEN N'Ключевая − 2.0 п.п.'
      WHEN ABS( (r.rate_liq * @scale_liq) - (k.key_rate_pct - 1.1) ) <= @eps_pp THEN N'Ключевая − 1.1 п.п.'
      WHEN ABS( (r.rate_liq * @scale_liq) - (k.key_rate_pct - 5.0) ) <= @eps_pp THEN N'Ключевая − 5.0 п.п.'
      ELSE NULL
    END AS rate_bucket
INTO #rc
FROM #rc0 r
JOIN #key_p k
  ON r.dt_rep >= k.dt_from AND r.dt_rep <= k.dt_to;

CREATE NONCLUSTERED INDEX IX_rc_dt_bucket ON #rc(dt_rep, rate_bucket) INCLUDE(out_rub, rate_use);

------------------------------------------------------------
-- 7) Панель 1: весь портфель (1 строка/день)
--    средняя только по тем, где rate_use IS NOT NULL
------------------------------------------------------------
IF OBJECT_ID('tempdb..#panel_all') IS NOT NULL DROP TABLE #panel_all;

SELECT
    r.dt_rep,
    SUM(r.out_rub) AS out_rub_total,
    SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub END) AS out_rub_with_rate,
    CAST(
      SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.rate_use * r.out_rub END)
      / NULLIF(SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub END),0)
      AS decimal(9,6)
    ) AS rate_con
INTO #panel_all
FROM #rc r
GROUP BY r.dt_rep;

------------------------------------------------------------
-- 8) Панель 2: бакеты (до 3 строк/день)
------------------------------------------------------------
IF OBJECT_ID('tempdb..#panel_bucket') IS NOT NULL DROP TABLE #panel_bucket;

SELECT
    r.dt_rep,
    r.rate_bucket,
    SUM(r.out_rub) AS out_rub_total,
    SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub END) AS out_rub_with_rate,
    CAST(
      SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.rate_use * r.out_rub END)
      / NULLIF(SUM(CASE WHEN r.rate_use IS NOT NULL THEN r.out_rub END),0)
      AS decimal(9,6)
    ) AS rate_con
INTO #panel_bucket
FROM #rc r
WHERE r.rate_bucket IS NOT NULL
GROUP BY r.dt_rep, r.rate_bucket;

------------------------------------------------------------
-- 9) Результаты
------------------------------------------------------------
-- Панель 1
SELECT dt_rep, out_rub_total, out_rub_with_rate, rate_con
FROM #panel_all
ORDER BY dt_rep;

-- Панель 2
SELECT dt_rep, rate_bucket, out_rub_total, out_rub_with_rate, rate_con
FROM #panel_bucket
ORDER BY dt_rep, rate_bucket;
