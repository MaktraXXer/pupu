
USE ALM;
SET NOCOUNT ON;

DECLARE @DateFrom date = '2026-11-15';
DECLARE @DateTo date = '2026-11-21';

DROP TABLE IF EXISTS #bal;
DROP TABLE IF EXISTS #rates;
DROP TABLE IF EXISTS #result;


/* 1. Баланс + ставка LIQUIDITY */

SELECT
    t.dt_rep,
    t.dt_open,
    t.con_id,

    CAST(t.out_rub AS decimal(20,2)) AS out_rub,
    CAST(t.rate_con AS decimal(9,4)) AS rate_balance,
    t.rate_con_src,

    dcr.rate AS rate_liq

INTO #bal

FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

OUTER APPLY (
    SELECT TOP (1)
        r.rate
    FROM LIQUIDITY.liq.DepositContract_Rate r WITH (NOLOCK)
    WHERE r.con_id = t.con_id
      AND (
          CASE
              WHEN t.dt_open = t.dt_rep
                  THEN DATEADD(day, 1, t.dt_rep)
              ELSE t.dt_rep
          END
      ) BETWEEN r.dt_from AND r.dt_to
    ORDER BY r.dt_from DESC, r.dt_to DESC
) dcr

WHERE t.dt_rep >= @DateFrom
  AND t.dt_rep < DATEADD(day, 3, @DateTo)

  AND t.section_name = N'Накопительный счёт'
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub IS NOT NULL

OPTION (MAXDOP 2, RECOMPILE);

CREATE CLUSTERED INDEX IX_bal
ON #bal (con_id, dt_rep);


/* 2. Рассчитываем ставку по алгоритму */

;WITH bal_pos AS (
    SELECT
        *,

        MIN(CASE
            WHEN rate_balance > 0
             AND rate_con_src = N'счет ультра,вручную'
                THEN rate_balance
        END) OVER (
            PARTITION BY con_id
            ORDER BY dt_rep
            ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING
        ) AS rate_pos

    FROM #bal
),

rate_calc AS (
    SELECT
        dt_rep,
        dt_open,
        out_rub,

        rate_balance,
        rate_liq,

        CAST(
            CASE
                WHEN rate_liq IS NULL
                    THEN CASE
                        WHEN rate_balance < 0
                            THEN COALESCE(rate_pos, rate_balance)
                        ELSE rate_balance
                    END

                WHEN rate_liq < 0 AND rate_balance > 0
                    THEN rate_balance

                WHEN rate_liq < 0 AND rate_balance < 0
                    THEN COALESCE(rate_pos, rate_balance)

                WHEN rate_liq >= 0 AND rate_balance >= 0
                    THEN rate_liq

                WHEN rate_liq > 0 AND rate_balance < 0
                    THEN rate_liq

                ELSE rate_liq
            END AS decimal(9,4)
        ) AS rate_use

    FROM bal_pos

    WHERE dt_rep BETWEEN @DateFrom AND @DateTo
)

SELECT *
INTO #rates
FROM rate_calc;


/* 3. Общие объемы и три средневзвешенные ставки */

;WITH aggr AS (
    SELECT
        r.dt_rep,
        s.data_scope,

        SUM(r.out_rub) AS out_rub_total,

        SUM(r.rate_balance * r.out_rub)
            / NULLIF(SUM(r.out_rub), 0) AS rate_source_a,

        SUM(r.rate_liq * r.out_rub)
            / NULLIF(SUM(r.out_rub), 0) AS rate_source_b,

        SUM(r.rate_use * r.out_rub)
            / NULLIF(SUM(r.out_rub), 0) AS rate_algorithm

    FROM #rates r

    CROSS APPLY (
        VALUES
            (N'портфель', 1),
            (N'новые',
                CASE WHEN r.dt_open = r.dt_rep
                    THEN 1 ELSE 0 END)
    ) s(data_scope, include_flag)

    WHERE s.include_flag = 1

    GROUP BY
        r.dt_rep,
        s.data_scope
),

calendar AS (
    SELECT DATEADD(day, n.n, @DateFrom) AS dt_rep
    FROM (
        VALUES (0),(1),(2),(3),(4),(5),(6)
    ) n(n)
)

SELECT
    c.dt_rep,
    s.data_scope,

    a.out_rub_total,
    a.rate_source_a,
    a.rate_source_b,
    a.rate_algorithm

INTO #result

FROM calendar c
CROSS JOIN (
    VALUES (N'портфель'), (N'новые')
) s(data_scope)

LEFT JOIN aggr a
    ON a.dt_rep = c.dt_rep
   AND a.data_scope = s.data_scope;


/* SELECT 1 — СТАВКА ИЗ БАЛАНСА */

SELECT
    dt_rep,
    data_scope,
    out_rub_total,
    rate_source_a AS rate_con

FROM #result
ORDER BY dt_rep, data_scope;


/* SELECT 2 — СТАВКА ИЗ LIQUIDITY */

SELECT
    dt_rep,
    data_scope,
    out_rub_total,
    rate_source_b AS rate_con

FROM #result
ORDER BY dt_rep, data_scope;


/* SELECT 3 — СТАВКА ПО НАШЕМУ АЛГОРИТМУ */

SELECT
    dt_rep,
    data_scope,
    out_rub_total,
    rate_algorithm AS rate_con

FROM #result
ORDER BY dt_rep, data_scope;
