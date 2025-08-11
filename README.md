USE [ALM_TEST];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2025-01-01';
DECLARE @DateTo   date = CAST(GETDATE() AS date);

DECLARE @DateFromWide date = DATEADD(day,-2,@DateFrom);
DECLARE @DateToWide   date = DATEADD(day, 2,@DateTo);

/* допуск сравнения в п.п. (на случай округления ставок) */
DECLARE @eps decimal(9,4) = 0.01;  -- 0.01 п.п.

/* ---------- базовый снимок + договорная ставка без фан-аута ---------- */
IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;

;WITH src AS (
    SELECT
          t.dt_rep
        , t.dt_open
        , t.con_id
        , CAST(t.out_rub  AS decimal(20,2)) AS out_rub
        , CAST(t.rate_con AS decimal(9,4))  AS rate_balance
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
    , dcr.rate AS rate_liq
INTO #bal
FROM src s
OUTER APPLY (
    SELECT TOP (1) r.rate
    FROM LIQUIDITY.liq.DepositContract_Rate r
    WHERE r.con_id = s.con_id
      AND s.eff_dt BETWEEN r.dt_from AND r.dt_to
    ORDER BY r.dt_from DESC, r.dt_to DESC
) dcr;

CREATE CLUSTERED INDEX IX_bal ON #bal (con_id, dt_rep);

/* ---------- выбор рабочей ставки (rate_use) ---------- */
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
             END AS decimal(9,4)
           ) AS rate_use
    FROM bal_pos
),
/* ---------- подтягиваем Ключевую (last-known <= dt_rep) и строим корзины ---------- */
rc_with_key AS (
    SELECT rc.*,
           kr.KEY_RATE,
           CASE
             WHEN rc.rate_liq IS NULL OR kr.KEY_RATE IS NULL THEN NULL
             WHEN ABS(rc.rate_liq - (kr.KEY_RATE - 2.0)) <= @eps THEN N'Ключевая − 2.0 п.п.'
             WHEN ABS(rc.rate_liq - (kr.KEY_RATE - 1.1)) <= @eps THEN N'Ключевая − 1.1 п.п.'
             WHEN ABS(rc.rate_liq - (kr.KEY_RATE - 5.0)) <= @eps THEN N'Ключевая − 5.0 п.п.'
             ELSE NULL
           END AS rate_bucket
    FROM rate_calc rc
    OUTER APPLY (
        SELECT TOP (1) k.KEY_RATE
        FROM ALM_TEST.price.key_rate_fact k WITH (NOLOCK)
        WHERE k.dt_rep <= rc.dt_rep
        ORDER BY k.dt_rep DESC
    ) kr
)

/* ---------- Панель 1: весь портфель (одна строка на день) ---------- */
SELECT
      rc.dt_rep
    , SUM(rc.out_rub) AS out_rub_total
    , CAST(SUM(rc.rate_use * rc.out_rub) / NULLIF(SUM(rc.out_rub),0) AS decimal(9,4)) AS rate_con
FROM rc_with_key rc
GROUP BY rc.dt_rep
ORDER BY rc.dt_rep;

/* ---------- Панель 2: разбивка по договорным корзинам относительно Ключевой ---------- */
SELECT
      rc.dt_rep
    , rc.rate_bucket
    , SUM(rc.out_rub) AS out_rub_total
    , CAST(SUM(rc.rate_use * rc.out_rub) / NULLIF(SUM(rc.out_rub),0) AS decimal(9,4)) AS rate_con
FROM rc_with_key rc
WHERE rc.rate_bucket IS NOT NULL
GROUP BY rc.dt_rep, rc.rate_bucket
ORDER BY rc.dt_rep, rc.rate_bucket;
