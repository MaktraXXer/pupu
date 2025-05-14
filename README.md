/*------------------------------------------------------------
  ТОП-20 клиентов по совокупному остатку
  Период: 23 янв 2025 – сегодня
------------------------------------------------------------*/

DECLARE @date_start  date = '2025-01-23';
DECLARE @date_end    date = CAST(GETDATE() AS date);
DECLARE @top_n       int  = 20;          -- размер рейтинга
GO         -- <-- отделяем блок DECLARE; дальше идёт CTE

/* 1. Суточный total-остаток клиента -------------------------*/
;WITH daily_balances AS (
    SELECT
        dt_rep,
        cli_id,
        SUM(out_rub) / 1000000.0  AS total_baln_mln  -- млн руб
    FROM ALM.ALM.balance_rest_all WITH (NOLOCK)
    WHERE dt_rep BETWEEN @date_start AND @date_end
      AND od_flag     = 1
      AND block_name  = N'Привлечение ФЛ'
      AND section_name NOT IN (N'Аккредитивы',
                               N'Аккредитив под строительство',
                               N'Брокерское обслуживание')
    GROUP BY dt_rep, cli_id
),

/* 2. Совокупный остаток за период ---------------------------*/
client_total AS (
    SELECT
        cli_id,
        SUM(total_baln_mln) AS sum_baln_mln
    FROM daily_balances
    GROUP BY cli_id
),

/* 3. Фиксируем пул ТОП-N клиентов ---------------------------*/
top_clients AS (
    SELECT TOP (@top_n)
           cli_id,
           sum_baln_mln
    FROM client_total
    ORDER BY sum_baln_mln DESC
),

/* 4. Остаток на начало / конец периода ---------------------*/
first_last AS (
    SELECT
        tc.cli_id,
        MAX(CASE WHEN db.dt_rep = @date_start THEN db.total_baln_mln END) AS start_baln_mln,
        MAX(CASE WHEN db.dt_rep = @date_end   THEN db.total_baln_mln END) AS end_baln_mln
    FROM top_clients tc
    JOIN daily_balances db ON db.cli_id = tc.cli_id
    GROUP BY tc.cli_id
)

/*------------------------------------------------------------
  >>> 1.  Рейтинг ТОП-20 по совокупному остатку
------------------------------------------------------------*/
SELECT
    ROW_NUMBER() OVER (ORDER BY t.sum_baln_mln DESC) AS rn,
    t.cli_id,
    ROUND(t.sum_baln_mln, 2) AS total_for_period_mln
FROM top_clients t
ORDER BY t.sum_baln_mln DESC;

GO   ---- отделяем след. запрос

/*------------------------------------------------------------
  >>> 2.  Остаток 23-01-2025 vs сегодня + дельта
------------------------------------------------------------*/
SELECT
    f.cli_id,
    ROUND(f.start_baln_mln, 2)                          AS balance_start_mln,
    ROUND(f.end_baln_mln,   2)                          AS balance_end_mln,
    ROUND(f.end_baln_mln - ISNULL(f.start_baln_mln,0),2) AS delta_mln
FROM first_last f
ORDER BY ABS(f.end_baln_mln - ISNULL(f.start_baln_mln,0)) DESC;

GO   ---- отделяем след. запрос

/*------------------------------------------------------------
  >>> 3.  Ежедневная динамика для этих же ТОП-20 (для BI/Excel)
------------------------------------------------------------*/
SELECT
    db.dt_rep,
    db.cli_id,
    ROUND(db.total_baln_mln, 2) AS total_baln_mln
FROM daily_balances db
JOIN top_clients   tc ON tc.cli_id = db.cli_id
ORDER BY db.dt_rep, db.cli_id;
