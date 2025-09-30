понял. Ниже — обновлённые скрипты: теперь календарь начинается включая @dt_rep = '2025-08-31' (а не с +1 дня). Поэтому:
	•	в амортизации вы увидите baseline на 2025-08-31 (полный портфель на дату отчёта);
	•	в выходах строка за 2025-08-31 тоже будет, но, учитывая фильтр dt_close > dt_rep, скорее всего там будут NULL (или 0), что логично.

⸻

Скрипт 1 — Амортизация (включая 2025-08-31)

DECLARE @dt_rep date = '2025-08-31';

WITH base AS (
    SELECT
        t.TSegmentname,
        CAST(t.dt_close AS date) AS dt_close_d,
        t.out_rub,
        t.rate_con,
        t.rate_trf
    FROM alm.[ALM].[vw_balance_rest_all] AS t WITH (NOLOCK)
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
    SELECT
        @dt_rep               AS d_start,   -- 👈 включаем дату отчёта
        MAX(dt_close_d)       AS d_end
    FROM base
),
calendar AS (
    SELECT d_start AS d
    FROM bounds
    UNION ALL
    SELECT DATEADD(day, 1, d)
    FROM calendar c
    JOIN bounds b ON c.d < b.d_end
),
segments AS (
    SELECT DISTINCT TSegmentname FROM base
),
grid AS (
    SELECT s.TSegmentname, c.d
    FROM segments s
    CROSS JOIN calendar c
),
active AS (
    SELECT
        g.d,
        g.TSegmentname,
        SUM(ISNULL(b.out_rub, 0)) AS out_rub_live,
        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END) AS trf_num,
        SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub END)              AS trf_den,
        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END) AS con_num,
        SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END)              AS con_den
    FROM grid g
    LEFT JOIN base b
      ON b.TSegmentname = g.TSegmentname
     AND b.dt_close_d   > g.d     -- включительно держим всё, что закрывается ПОСЛЕ даты d
    GROUP BY g.d, g.TSegmentname
)
SELECT
    a.d                              AS [date],
    a.TSegmentname                   AS tsegmentname,
    a.out_rub_live                   AS out_rub,
    CAST(a.trf_num / NULLIF(a.trf_den, 0) AS DECIMAL(12,6)) AS rate_trf_srvz,
    CAST(a.con_num / NULLIF(a.con_den, 0) AS DECIMAL(12,6)) AS rate_con_srvz
FROM active a
ORDER BY a.d, a.TSegmentname
OPTION (MAXRECURSION 0);


⸻

Скрипт 2 — Выходы (включая 2025-08-31)

DECLARE @dt_rep date = '2025-08-31';

WITH base AS (
    SELECT
        t.TSegmentname,
        CAST(t.dt_close AS date) AS dt_close_d,
        t.out_rub,
        t.rate_con,
        t.rate_trf
    FROM alm.[ALM].[vw_balance_rest_all] AS t WITH (NOLOCK)
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
    SELECT
        @dt_rep         AS d_start,  -- 👈 включаем дату отчёта
        MAX(dt_close_d) AS d_end
    FROM base
),
calendar AS (
    SELECT d_start AS d
    FROM bounds
    UNION ALL
    SELECT DATEADD(day, 1, d)
    FROM calendar c
    JOIN bounds b ON c.d < b.d_end
),
segments AS (
    SELECT DISTINCT TSegmentname FROM base
),
grid AS (
    SELECT s.TSegmentname, c.d
    FROM segments s
    CROSS JOIN calendar c
),
closings AS (
    SELECT
        dt_close_d AS d,
        TSegmentname,
        SUM(out_rub)                                                    AS out_rub_close,
        SUM(CASE WHEN rate_trf IS NOT NULL THEN out_rub * rate_trf END) AS trf_num,
        SUM(CASE WHEN rate_trf IS NOT NULL THEN out_rub END)            AS trf_den,
        SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub * rate_con END) AS con_num,
        SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END)            AS con_den
    FROM base
    GROUP BY dt_close_d, TSegmentname
)
SELECT
    g.d                               AS [date],
    g.TSegmentname                    AS tsegmentname,
    c.out_rub_close                   AS out_rub,
    CAST(c.trf_num / NULLIF(c.trf_den, 0) AS DECIMAL(12,6)) AS rate_trf_srvz,
    CAST(c.con_num / NULLIF(c.con_den, 0) AS DECIMAL(12,6)) AS rate_con_srvz
FROM grid g
LEFT JOIN closings c
  ON c.TSegmentname = g.TSegmentname
 AND c.d            = g.d
ORDER BY g.d, g.TSegmentname
OPTION (MAXRECURSION 0);

Если нужно, могу добавить агрегированную «Итого по двум сегментам» строку на каждую дату.
