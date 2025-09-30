DECLARE @dt_rep date = '2025-08-31';

WITH base AS (
    SELECT
        t.TSegmentname,
        CAST(t.dt_close AS date) AS dt_close_d,
        t.out_rub,
        t.rate_con,
        t.rate_trf
    FROM alm.[ALM].[vw_balance_rest_all] t WITH (NOLOCK)
    WHERE t.dt_rep       = @dt_rep
      AND t.section_name = N'Срочные'
      AND t.block_name   = N'Привлечение ЮЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND t.AP           = N'Пассив'
      AND t.dt_close     > t.dt_rep
      AND t.out_rub IS NOT NULL
      AND t.tprod_name   = N'Депозиты ЮЛ'
      AND t.TSegmentname IN (N'Ипотека', N'Розничный бизнес')
),
bounds AS (
    SELECT @dt_rep AS d_start, ISNULL(MAX(dt_close_d), @dt_rep) AS d_end
    FROM base
),
cal AS (
    -- календарь от @dt_rep до последнего dt_close
    SELECT d_start AS d FROM bounds
    UNION ALL
    SELECT DATEADD(day, 1, c.d)
    FROM cal c
    JOIN bounds b ON c.d < b.d_end
),
segments AS (
    SELECT DISTINCT TSegmentname FROM base
),
all_dates AS (
    -- строки на каждую дату по каждому сегменту (чтобы дни без событий тоже были)
    SELECT s.TSegmentname, c.d
    FROM segments s
    CROSS JOIN cal c
),
closings AS (
    -- что закрывается в каждую дату, + части для взвешенных ставок
    SELECT
        b.TSegmentname,
        b.dt_close_d AS d,
        SUM(b.out_rub)                                                    AS out_rub_close,
        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END) AS trf_num,
        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub END)              AS trf_den,
        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END) AS con_num,
        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END)              AS con_den
    FROM base b
    GROUP BY b.TSegmentname, b.dt_close_d
),
series AS (
    -- присоединяем закрытия (где пусто — берём 0 для кумулятивов)
    SELECT
        ad.TSegmentname,
        ad.d,
        ISNULL(c.out_rub_close, 0) AS out_close0,
        ISNULL(c.trf_num,      0) AS trf_num0,
        ISNULL(c.trf_den,      0) AS trf_den0,
        ISNULL(c.con_num,      0) AS con_num0,
        ISNULL(c.con_den,      0) AS con_den0
    FROM all_dates ad
    LEFT JOIN closings c
      ON c.TSegmentname = ad.TSegmentname
     AND c.d            = ad.d
),
cum AS (
    -- кумулятив закрытий до и включая дату d
    SELECT
        s.TSegmentname,
        s.d,
        SUM(s.out_close0) OVER (PARTITION BY s.TSegmentname ORDER BY s.d ROWS UNBOUNDED PRECEDING) AS cum_out,
        SUM(s.trf_num0)   OVER (PARTITION BY s.TSegmentname ORDER BY s.d ROWS UNBOUNDED PRECEDING) AS cum_trf_num,
        SUM(s.trf_den0)   OVER (PARTITION BY s.TSegmentname ORDER BY s.d ROWS UNBOUNDED PRECEDING) AS cum_trf_den,
        SUM(s.con_num0)   OVER (PARTITION BY s.TSegmentname ORDER BY s.d ROWS UNBOUNDED PRECEDING) AS cum_con_num,
        SUM(s.con_den0)   OVER (PARTITION BY s.TSegmentname ORDER BY s.d ROWS UNBOUNDED PRECEDING) AS cum_con_den
    FROM series s
),
init AS (
    -- стартовые суммы на @dt_rep
    SELECT
        b.TSegmentname,
        SUM(b.out_rub) AS init_out,
        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END) AS init_trf_num,
        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub END)               AS init_trf_den,
        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END)  AS init_con_num,
        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END)               AS init_con_den
    FROM base b
    GROUP BY b.TSegmentname
)
SELECT
    c.d                         AS [date],
    c.TSegmentname              AS tsegmentname,
    (i.init_out - c.cum_out)    AS out_rub,
    CAST( (i.init_trf_num - c.cum_trf_num) / NULLIF(i.init_trf_den - c.cum_trf_den, 0) AS DECIMAL(12,6) ) AS rate_trf_srvz,
    CAST( (i.init_con_num - c.cum_con_num) / NULLIF(i.init_con_den - c.cum_con_den, 0) AS DECIMAL(12,6) ) AS rate_con_srvz
FROM cum c
JOIN init i ON i.TSegmentname = c.TSegmentname
ORDER BY c.d, c.TSegmentname
OPTION (MAXRECURSION 0);
