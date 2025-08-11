USE [ALM_TEST];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2025-01-01';
DECLARE @DateTo   date = CAST(GETDATE() AS date);

DECLARE @DateFromWide date = DATEADD(day,-2,@DateFrom); -- look-around
DECLARE @DateToWide   date = DATEADD(day, 2,@DateTo);

/* Допуск для сравнения в п.п. (после нормализации в проценты): */
DECLARE @eps_pp decimal(9,4) = 0.01;  -- 0.01 п.п.

/*===========================================================
  1) База: баланс + договорная ставка (без фан-аута)
     Робастный джойн по con_id и интервалам дат + фоллбэк
===========================================================*/
IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;

;WITH src AS (
    SELECT
          t.dt_rep
        , t.dt_open
        , t.con_id
        , CAST(t.out_rub  AS decimal(20,2)) AS out_rub
        , CAST(t.rate_con AS decimal(9,6))  AS rate_balance
        , t.rate_con_src
        , CASE WHEN t.dt_open = t.dt_rep
               THEN DATEADD(day,1,t.dt_rep)       -- «нулевой» день → +1
               ELSE t.dt_rep
          END AS eff_dt
    FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
    WHERE t.dt_rep BETWEEN @DateFromWide AND @DateToWide
      AND t.section_name = N'Накопительный счёт'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND t.prod_id      = '3103'
      AND t.out_rub IS NOT NULL
)
SELECT
      s.dt_rep
    , s.dt_open
    , s.con_id
    , s.out_rub
    , s.rate_balance
    , s.rate_con_src
    , COALESCE(d_on.rate, d_prev.rate) AS rate_liq  -- договорная из LIQUIDITY
INTO #bal
FROM src s
OUTER APPLY (
    /* 1) Ставка, активная в eff_dt (инклюзивный низ, инклюзивный верх + учёт NULL dt_to) */
    SELECT TOP (1) r.rate
    FROM LIQUIDITY.liq.DepositContract_Rate r
    WHERE CONVERT(varchar(64), r.con_id) = CONVERT(varchar(64), s.con_id)
      AND r.dt_from <= DATEADD(second, 86399, CAST(s.eff_dt AS datetime))
      AND (r.dt_to IS NULL OR r.dt_to >= CAST(s.eff_dt AS datetime))
    ORDER BY r.dt_from DESC, ISNULL(r.dt_to,'9999-12-31') DESC
) d_on
OUTER APPLY (
    /* 2) Фоллбэк: последняя ставка до eff_dt, если активную не нашли */
    SELECT TOP (1) r.rate
    FROM LIQUIDITY.liq.DepositContract_Rate r
    WHERE CONVERT(varchar(64), r.con_id) = CONVERT(varchar(64), s.con_id)
      AND r.dt_from <= CAST(s.eff_dt AS datetime)
    ORDER BY r.dt_from DESC
) d_prev;

CREATE CLUSTERED INDEX IX_bal ON #bal (con_id, dt_rep);

/*===========================================================
  2) Рабочая ставка rate_use (как в твоей процедуре)
===========================================================*/
;WITH bal_pos AS (
    SELECT  b.*,
            MIN(CASE WHEN b.rate_balance > 0
                      AND b.rate_con_src = N'счет ультра,вручную'
                     THEN b.rate_balance END)
                OVER (PARTITION BY b.con_id
                      ORDER BY b.dt_rep
                      ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
    FROM #bal b
    WHERE b.dt_rep BETWEEN @DateFrom AND @DateTo
),
rate_calc AS (
    SELECT *,
           CAST(
             CASE
               WHEN rate_liq IS NULL THEN
                    CASE WHEN rate_balance < 0
                         THEN COALESCE(rate_pos, rate_balance)
                         ELSE rate_balance
                    END
               WHEN rate_liq < 0  AND rate_balance > 0 THEN rate_balance
               WHEN rate_liq < 0  AND rate_balance < 0 THEN COALESCE(rate_pos, rate_balance)
               WHEN rate_liq >= 0 AND rate_balance >= 0 THEN rate_liq
               WHEN rate_liq > 0  AND rate_balance  < 0 THEN rate_liq
               ELSE rate_liq
             END AS decimal(9,6)
           ) AS rate_use
    FROM bal_pos
)
SELECT *
INTO #rc0
FROM rate_calc;

CREATE INDEX IX_rc0_dt ON #rc0 (dt_rep) INCLUDE (out_rub, rate_use, rate_liq);

/*===========================================================
  3) Ключевая ставка (last-known ≤ dt_rep) + нормализация шкал
     Авто-детект: если среднее < 1 → это доли, умножаем на 100.
===========================================================*/
DECLARE @kr_avg decimal(18,6) =
(SELECT AVG(CAST(KEY_RATE AS decimal(18,6)))
   FROM ALM_TEST.price.key_rate_fact WITH (NOLOCK)
  WHERE dt_rep BETWEEN DATEADD(day,-30,@DateFrom) AND @DateTo);

DECLARE @kr_scale decimal(12,6) = CASE WHEN @kr_avg IS NOT NULL AND @kr_avg < 1 THEN 100.0 ELSE 1.0 END;

DECLARE @liq_avg decimal(18,6) =
(SELECT AVG(CAST(ABS(rate_liq) AS decimal(18,6))) FROM #rc0 WHERE rate_liq IS NOT NULL);

DECLARE @liq_scale decimal(12,6) = CASE WHEN @liq_avg IS NOT NULL AND @liq_avg < 1 THEN 100.0 ELSE 1.0 END;

/* last-known ключевая + бакеты по ДОГОВОРНОЙ ставке */
SELECT
      rc.*
    , kr.KEY_RATE
    , CAST(kr.KEY_RATE * @kr_scale AS decimal(18,6))  AS key_rate_pct
    , CAST(rc.rate_liq * @liq_scale AS decimal(18,6)) AS rate_liq_pct
    , CASE
        WHEN rc.rate_liq IS NULL OR kr.KEY_RATE IS NULL THEN NULL
        WHEN ABS( (rc.rate_liq * @liq_scale) - (kr.KEY_RATE * @kr_scale - 2.0) )  <= @eps_pp THEN N'Ключевая − 2.0 п.п.'
        WHEN ABS( (rc.rate_liq * @liq_scale) - (kr.KEY_RATE * @kr_scale - 1.1) )  <= @eps_pp THEN N'Ключевая − 1.1 п.п.'
        WHEN ABS( (rc.rate_liq * @liq_scale) - (kr.KEY_RATE * @kr_scale - 5.0) )  <= @eps_pp THEN N'Ключевая − 5.0 п.п.'
        ELSE NULL
      END AS rate_bucket
INTO #rc
FROM #rc0 rc
OUTER APPLY (
    SELECT TOP (1) k.KEY_RATE
    FROM ALM_TEST.price.key_rate_fact k WITH (NOLOCK)
    WHERE k.dt_rep <= rc.dt_rep
    ORDER BY k.dt_rep DESC
) kr;

CREATE INDEX IX_rc_dt_bucket ON #rc (dt_rep, rate_bucket) INCLUDE (out_rub, rate_use);

/*===========================================================
  4) Панель 1: весь портфель (1 строка на день)
     Средняя — только по тем, где rate_use не NULL
===========================================================*/
IF OBJECT_ID('tempdb..#panel_all') IS NOT NULL DROP TABLE #panel_all;

SELECT
      rc.dt_rep
    , SUM(rc.out_rub)                                                   AS out_rub_total
    , SUM(CASE WHEN rc.rate_use IS NOT NULL THEN rc.out_rub END)        AS out_rub_with_rate
    , CAST(
        SUM(CASE WHEN rc.rate_use IS NOT NULL THEN rc.rate_use * rc.out_rub END)
        / NULLIF(SUM(CASE WHEN rc.rate_use IS NOT NULL THEN rc.out_rub END),0)
      AS decimal(9,6))                                                  AS rate_con
INTO #panel_all
FROM #rc rc
GROUP BY rc.dt_rep;

CREATE INDEX IX_panel_all ON #panel_all (dt_rep);

/*===========================================================
  5) Панель 2: по бакетам (до 3 строк на день)
===========================================================*/
IF OBJECT_ID('tempdb..#panel_bucket') IS NOT NULL DROP TABLE #panel_bucket;

SELECT
      rc.dt_rep
    , rc.rate_bucket
    , SUM(rc.out_rub)                                                   AS out_rub_total
    , SUM(CASE WHEN rc.rate_use IS NOT NULL THEN rc.out_rub END)        AS out_rub_with_rate
    , CAST(
        SUM(CASE WHEN rc.rate_use IS NOT NULL THEN rc.rate_use * rc.out_rub END)
        / NULLIF(SUM(CASE WHEN rc.rate_use IS NOT NULL THEN rc.out_rub END),0)
      AS decimal(9,6))                                                  AS rate_con
INTO #panel_bucket
FROM #rc rc
WHERE rc.rate_bucket IS NOT NULL
GROUP BY rc.dt_rep, rc.rate_bucket;

CREATE INDEX IX_panel_bucket ON #panel_bucket (dt_rep, rate_bucket);

/*===========================================================
  6) РЕЗУЛЬТАТЫ
===========================================================*/
-- Панель 1: весь портфель
SELECT dt_rep, out_rub_total, out_rub_with_rate, rate_con
FROM #panel_all
ORDER BY dt_rep;

-- Панель 2: по бакетам
SELECT dt_rep, rate_bucket, out_rub_total, out_rub_with_rate, rate_con
FROM #panel_bucket
ORDER BY dt_rep, rate_bucket;

/*===========================================================
  7) (опционально) Диагностика покрытия ставки из LIQUIDITY
     — сколько NULL по дням и примеры договоров
===========================================================*/
-- Кол-во договоров без rate_liq по дням:
-- SELECT dt_rep, COUNT(*) AS cnt_total,
--        SUM(CASE WHEN rate_liq IS NULL THEN 1 ELSE 0 END) AS cnt_no_rate_liq
-- FROM #rc
-- GROUP BY dt_rep
-- ORDER BY dt_rep;

-- Примеры con_id без ставки:
-- SELECT TOP (50) dt_rep, con_id, rate_balance, rate_liq
-- FROM #rc
-- WHERE rate_liq IS NULL
-- ORDER BY dt_rep DESC;
