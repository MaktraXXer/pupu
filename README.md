/* СВЕРКА КОРЗИН ПО МЕСЯЦУ ОТКРЫТИЯ (на основе финальной дневной ленты) */
SET NOCOUNT ON;

IF OBJECT_ID('tempdb..#bal_prom') IS NULL
   OR OBJECT_ID('tempdb..#daily_pre') IS NULL
   OR OBJECT_ID('tempdb..#daily_base_adj') IS NULL
   OR OBJECT_ID('tempdb..#firstday_assigned') IS NULL
BEGIN
    RAISERROR(N'Нет промежуточных таблиц Части 2. Сначала выполните Часть 2.',16,1);
    RETURN;
END;

/* 1) финальная дневная лента БЕЗ агрегации по дням (с учётом base-day и 1-го числа) */
WITH final_daily AS (
    -- все дни, кроме тех base-day, которые заменяем:
    SELECT dp.con_id, dp.cli_id, dp.dt_rep, dp.out_rub, dp.rate_con
    FROM   #daily_pre dp
    WHERE  NOT EXISTS (
              SELECT 1 FROM #base_dates bd
              WHERE  bd.cli_id = dp.cli_id AND bd.dt_rep = dp.dt_rep
           )
    UNION ALL
    -- base-day: уже переложенный объём на победителя (одна строка на клиента)
    SELECT con_id, cli_id, dt_rep, out_rub, rate_con
    FROM   #daily_base_adj
    UNION ALL
    -- 1-е числа: полный перелив на победителя (одна строка на клиента)
    SELECT con_id, cli_id, dt_rep, out_rub, rate_con
    FROM   #firstday_assigned
),
tagged AS (
    /* добавляем месяц открытия и месяц отчётной даты */
    SELECT
        fd.cli_id,
        fd.con_id,
        fd.dt_rep,
        fd.out_rub,
        fd.rate_con,
        bp.dt_open,
        m_this = DATEFROMPARTS(YEAR(fd.dt_rep), MONTH(fd.dt_rep), 1),
        m_open = DATEFROMPARTS(YEAR(bp.dt_open), MONTH(bp.dt_open), 1)
    FROM final_daily fd
    JOIN #bal_prom  bp ON bp.con_id = fd.con_id
),
flags AS (
    /* для каждого клиента и дня — есть ли у него открытия в текущем / предыдущем месяцах */
    SELECT
        t.*,
        m_prev  = DATEADD(month,-1, m_this),
        has_cur = MAX(CASE WHEN t.m_open = t.m_this THEN 1 ELSE 0 END)
                   OVER (PARTITION BY t.cli_id, t.dt_rep),
        has_prev = MAX(CASE WHEN t.m_open = DATEADD(month,-1,t.m_this) THEN 1 ELSE 0 END)
                    OVER (PARTITION BY t.cli_id, t.dt_rep)
    FROM tagged t
),
buckets AS (
    SELECT
        dt_rep,
        bucket =
          CASE
            WHEN m_open = m_this THEN N'Открыты_в_текущем_мес'
            WHEN m_open = m_prev AND has_cur = 0 THEN N'Открыты_в_пред_мес_без_текущих'
            WHEN m_open = m_prev AND has_cur = 1 THEN N'Открыты_в_пред_мес_с_текущими'
            ELSE N'ПРОЧЕЕ'
          END,
        out_rub,
        rate_con,
        cli_id,
        has_prev
    FROM flags
)
-- 2а) три основные корзины
SELECT
    dt_rep,
    bucket,
    out_rub_total = SUM(out_rub),
    rate_avg      = SUM(CAST(out_rub AS decimal(38,6))*CAST(rate_con AS decimal(38,6)))
                    / NULLIF(SUM(CAST(out_rub AS decimal(38,6))),0)
FROM buckets
WHERE bucket <> N'ПРОЧЕЕ'
GROUP BY dt_rep, bucket

UNION ALL

-- 2б) «ставка по счетам открытым в текущем месяце» ТОЛЬКО для той когорты,
--     у которой есть ещё и открытия в предыдущем месяце
SELECT
    f.dt_rep,
    N'Открыты_в_текущем_мес_(кохорта_с_пред_мес)' AS bucket,
    SUM(f.out_rub) AS out_rub_total,
    SUM(CAST(f.out_rub AS decimal(38,6))*CAST(f.rate_con AS decimal(38,6)))
      / NULLIF(SUM(CAST(f.out_rub AS decimal(38,6))),0) AS rate_avg
FROM flags f
WHERE f.m_open = f.m_this
  AND f.has_prev = 1
GROUP BY f.dt_rep
ORDER BY dt_rep, bucket;
