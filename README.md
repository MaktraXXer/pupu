/*-----------------------------------------------------------
  ТОП-20 клиентов по совокупному остатку
  Период: с 23 января 2025 г. по «сегодня»
  Платформа: MS SQL Server
-----------------------------------------------------------*/

DECLARE @date_start  date = '2025-01-23';        -- начало интервала
DECLARE @date_end    date = CONVERT(date, GETDATE());  -- сегодня
DECLARE @top_n       int  = 20;                  -- сколько клиентов берём в TOP-N

/* 1. Суточный total-остаток каждого клиента  ----------------*/
;WITH daily_balances AS (
    SELECT
        dt_rep,                    -- дата отчёта
        cli_id,                    -- клиент
        SUM(out_rub) / 1_000_000.0 AS total_baln_mln  -- остаток, млн руб
    FROM ALM.ALM.balance_rest_all WITH (NOLOCK)
    WHERE dt_rep BETWEEN @date_start AND @date_end
      AND od_flag    = 1
      AND block_name = N'Привлечение ФЛ'
      AND section_name NOT IN (N'Аккредитивы',
                               N'Аккредитив под строительство',
                               N'Брокерское обслуживание')
    GROUP BY dt_rep, cli_id
),

/* 2. Совокупный остаток за период на клиента ----------------*/
client_total AS (
    SELECT
        cli_id,
        SUM(total_baln_mln) AS sum_baln_mln   -- тотал за весь период
    FROM daily_balances
    GROUP BY cli_id
),

/* 3. Фиксируем пул ТОП-N клиентов ---------------------------*/
top_clients AS (
    SELECT TOP (@top_n)  cli_id, sum_baln_mln
    FROM client_total
    ORDER BY sum_baln_mln DESC
),

/* 4. Баланс «старт / финиш» и дельта ------------------------*/
first_last AS (
    SELECT
        tc.cli_id,

        MAX(CASE WHEN db.dt_rep = @date_start THEN db.total_baln_mln END) AS start_baln_mln,
        MAX(CASE WHEN db.dt_rep = @date_end   THEN db.total_baln_mln END) AS end_baln_mln
    FROM top_clients    tc
    JOIN daily_balances db ON db.cli_id = tc.cli_id
    GROUP BY tc.cli_id
)

/*-----------------------------------------------------------
  >>> Итог 1: перечень ТОП-20 + суммарный остаток за период
-----------------------------------------------------------*/
SELECT
    ROW_NUMBER() OVER (ORDER BY t.sum_baln_mln DESC) AS rn,
    t.cli_id,
    ROUND(t.sum_baln_mln, 2)   AS total_for_period_mln
FROM top_clients t
ORDER BY t.sum_baln_mln DESC;


/*-----------------------------------------------------------
  >>> Итог 2: старт / финиш / дельта для тех же клиентов
-----------------------------------------------------------*/
SELECT
    f.cli_id,
    ROUND(f.start_baln_mln, 2)        AS balance_start_mln,
    ROUND(f.end_baln_mln,   2)        AS balance_end_mln,
    ROUND(f.end_baln_mln - COALESCE(f.start_baln_mln,0), 2) AS delta_mln
FROM first_last f
ORDER BY ABS(f.end_baln_mln - COALESCE(f.start_baln_mln,0)) DESC;


/*-----------------------------------------------------------
  >>> Итог 3: суточная динамика по каждому из ТОП-20
              (можно отправлять в BI / Excel)
-----------------------------------------------------------*/
SELECT
    db.dt_rep,
    db.cli_id,
    ROUND(db.total_baln_mln, 2) AS total_baln_mln
FROM daily_balances db
JOIN top_clients   tc ON tc.cli_id = db.cli_id
ORDER BY db.dt_rep, db.cli_id;
