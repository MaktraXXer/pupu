DECLARE @dt_rep date = '2026-06-30';


/* ============================================================
   1. ИСХОДНЫЙ ИПОТЕЧНЫЙ ПОРТФЕЛЬ
   ============================================================ */

IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;

SELECT
    CAST(t.dt_close AS date)               AS dt_close_d,
    ISNULL(t.is_floatrate, 0)              AS is_floatrate,
    ABS(t.out_rub)                         AS out_rub,
    t.rate_int,
    t.rate_trf
INTO #base
FROM ALM.[ALM].[balance_rest_all] t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep
  AND t.BLOCK_NAME   = N'Кредиты ФЛ'
  AND t.SECTION_NAME = N'Ипотека'
  AND t.od_flag      = 1
  AND t.dt_close     > @dt_rep
  AND t.out_rub IS NOT NULL;


/* ============================================================
   2. КАЛЕНДАРЬ
   ============================================================ */

DECLARE @d_end date;

SELECT @d_end = MAX(dt_close_d)
FROM #base;

SET @d_end = ISNULL(@d_end, @dt_rep);


IF OBJECT_ID('tempdb..#calendar') IS NOT NULL DROP TABLE #calendar;

;WITH N AS
(
    SELECT TOP (
        CASE
            WHEN DATEDIFF(DAY, @dt_rep, @d_end) + 1 >= 1
                THEN DATEDIFF(DAY, @dt_rep, @d_end) + 1
            ELSE 1
        END
    )
        ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS n
    FROM sys.all_objects a
    CROSS JOIN sys.all_objects b
)
SELECT
    DATEADD(DAY, n, @dt_rep) AS d
INTO #calendar
FROM N;


/* ============================================================
   3. ГРУППЫ FIX / FLOAT
   ============================================================ */

IF OBJECT_ID('tempdb..#groups') IS NOT NULL DROP TABLE #groups;

SELECT DISTINCT
    is_floatrate
INTO #groups
FROM #base;


IF OBJECT_ID('tempdb..#grid') IS NOT NULL DROP TABLE #grid;

SELECT
    g.is_floatrate,
    c.d
INTO #grid
FROM #groups g
CROSS JOIN #calendar c;


/* ============================================================
   4. ПЛАНОВЫЕ ВЫХОДЫ ПО ДАТАМ
   ============================================================ */

IF OBJECT_ID('tempdb..#closings') IS NOT NULL DROP TABLE #closings;

SELECT
    b.dt_close_d AS d,
    b.is_floatrate,

    SUM(b.out_rub) AS out_rub_close,

    SUM(
        CASE
            WHEN b.rate_int IS NOT NULL
                THEN b.out_rub * b.rate_int
        END
    ) AS int_num,

    SUM(
        CASE
            WHEN b.rate_int IS NOT NULL
                THEN b.out_rub
        END
    ) AS int_den,

    SUM(
        CASE
            WHEN b.rate_trf IS NOT NULL
                THEN b.out_rub * b.rate_trf
        END
    ) AS trf_num,

    SUM(
        CASE
            WHEN b.rate_trf IS NOT NULL
                THEN b.out_rub
        END
    ) AS trf_den

INTO #closings
FROM #base b
GROUP BY
    b.dt_close_d,
    b.is_floatrate;


/* ============================================================
   5. НАКОПЛЕННЫЕ ВЫХОДЫ
   ============================================================ */

IF OBJECT_ID('tempdb..#closings_cum') IS NOT NULL DROP TABLE #closings_cum;

SELECT
    c.is_floatrate,
    c.d,

    SUM(c.out_rub_close) OVER (
        PARTITION BY c.is_floatrate
        ORDER BY c.d
        ROWS UNBOUNDED PRECEDING
    ) AS cum_out_rub,

    SUM(c.int_num) OVER (
        PARTITION BY c.is_floatrate
        ORDER BY c.d
        ROWS UNBOUNDED PRECEDING
    ) AS cum_int_num,

    SUM(c.int_den) OVER (
        PARTITION BY c.is_floatrate
        ORDER BY c.d
        ROWS UNBOUNDED PRECEDING
    ) AS cum_int_den,

    SUM(c.trf_num) OVER (
        PARTITION BY c.is_floatrate
        ORDER BY c.d
        ROWS UNBOUNDED PRECEDING
    ) AS cum_trf_num,

    SUM(c.trf_den) OVER (
        PARTITION BY c.is_floatrate
        ORDER BY c.d
        ROWS UNBOUNDED PRECEDING
    ) AS cum_trf_den

INTO #closings_cum
FROM #closings c;


/* ============================================================
   6. ИСХОДНЫЙ ПОРТФЕЛЬ
   ============================================================ */

IF OBJECT_ID('tempdb..#init') IS NOT NULL DROP TABLE #init;

SELECT
    b.is_floatrate,

    SUM(b.out_rub) AS init_out,

    SUM(
        CASE
            WHEN b.rate_int IS NOT NULL
                THEN b.out_rub * b.rate_int
        END
    ) AS init_int_num,

    SUM(
        CASE
            WHEN b.rate_int IS NOT NULL
                THEN b.out_rub
        END
    ) AS init_int_den,

    SUM(
        CASE
            WHEN b.rate_trf IS NOT NULL
                THEN b.out_rub * b.rate_trf
        END
    ) AS init_trf_num,

    SUM(
        CASE
            WHEN b.rate_trf IS NOT NULL
                THEN b.out_rub
        END
    ) AS init_trf_den

INTO #init
FROM #base b
GROUP BY
    b.is_floatrate;


/* ============================================================
   7. АМОРТИЗАЦИЯ
   dt_close = d уже НЕ входит в остаток на дату d
   ============================================================ */

SELECT
    g.d AS [date],

    g.is_floatrate,

    CASE
        WHEN g.is_floatrate = 1 THEN N'Плавающая'
        ELSE N'Фиксированная'
    END AS rate_type,

    i.init_out
        - ISNULL(cc.cum_out_rub, 0) AS out_rub,

    CAST(
        (
            i.init_int_num
            - ISNULL(cc.cum_int_num, 0)
        )
        /
        NULLIF(
            i.init_int_den
            - ISNULL(cc.cum_int_den, 0),
            0
        )
        AS DECIMAL(12,6)
    ) AS rate_int_srvz,

    CAST(
        (
            i.init_trf_num
            - ISNULL(cc.cum_trf_num, 0)
        )
        /
        NULLIF(
            i.init_trf_den
            - ISNULL(cc.cum_trf_den, 0),
            0
        )
        AS DECIMAL(12,6)
    ) AS rate_trf_srvz

FROM #grid g

JOIN #init i
    ON i.is_floatrate = g.is_floatrate

OUTER APPLY
(
    SELECT TOP (1)
        *
    FROM #closings_cum x
    WHERE x.is_floatrate = g.is_floatrate
      AND x.d <= g.d
    ORDER BY x.d DESC
) cc

ORDER BY
    g.d,
    g.is_floatrate;





DECLARE @dt_rep date = '2026-06-30';


/* ============================================================
   1. ИСХОДНЫЙ ИПОТЕЧНЫЙ ПОРТФЕЛЬ
   ============================================================ */

IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;

SELECT
    CAST(t.dt_close AS date)               AS dt_close_d,
    ISNULL(t.is_floatrate, 0)              AS is_floatrate,
    ABS(t.out_rub)                         AS out_rub,
    t.rate_int,
    t.rate_trf
INTO #base
FROM ALM.[ALM].[balance_rest_all] t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep
  AND t.BLOCK_NAME   = N'Кредиты ФЛ'
  AND t.SECTION_NAME = N'Ипотека'
  AND t.od_flag      = 1
  AND t.dt_close     > @dt_rep
  AND t.out_rub IS NOT NULL;


/* ============================================================
   2. КАЛЕНДАРЬ
   ============================================================ */

DECLARE @d_end date;

SELECT @d_end = MAX(dt_close_d)
FROM #base;

SET @d_end = ISNULL(@d_end, @dt_rep);


IF OBJECT_ID('tempdb..#calendar') IS NOT NULL DROP TABLE #calendar;

;WITH N AS
(
    SELECT TOP (
        CASE
            WHEN DATEDIFF(DAY, @dt_rep, @d_end) + 1 >= 1
                THEN DATEDIFF(DAY, @dt_rep, @d_end) + 1
            ELSE 1
        END
    )
        ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS n
    FROM sys.all_objects a
    CROSS JOIN sys.all_objects b
)
SELECT
    DATEADD(DAY, n, @dt_rep) AS d
INTO #calendar
FROM N;


/* ============================================================
   3. FIX / FLOAT
   ============================================================ */

IF OBJECT_ID('tempdb..#groups') IS NOT NULL DROP TABLE #groups;

SELECT DISTINCT
    is_floatrate
INTO #groups
FROM #base;


IF OBJECT_ID('tempdb..#grid') IS NOT NULL DROP TABLE #grid;

SELECT
    g.is_floatrate,
    c.d
INTO #grid
FROM #groups g
CROSS JOIN #calendar c;


/* ============================================================
   4. ВЫХОДЫ
   ============================================================ */

IF OBJECT_ID('tempdb..#closings') IS NOT NULL DROP TABLE #closings;

SELECT
    b.dt_close_d AS d,
    b.is_floatrate,

    SUM(b.out_rub) AS out_rub_close,

    SUM(
        CASE
            WHEN b.rate_int IS NOT NULL
                THEN b.out_rub * b.rate_int
        END
    ) AS int_num,

    SUM(
        CASE
            WHEN b.rate_int IS NOT NULL
                THEN b.out_rub
        END
    ) AS int_den,

    SUM(
        CASE
            WHEN b.rate_trf IS NOT NULL
                THEN b.out_rub * b.rate_trf
        END
    ) AS trf_num,

    SUM(
        CASE
            WHEN b.rate_trf IS NOT NULL
                THEN b.out_rub
        END
    ) AS trf_den

INTO #closings
FROM #base b
GROUP BY
    b.dt_close_d,
    b.is_floatrate;


/* ============================================================
   5. РЕЗУЛЬТАТ
   ============================================================ */

SELECT
    g.d AS [date],

    g.is_floatrate,

    CASE
        WHEN g.is_floatrate = 1 THEN N'Плавающая'
        ELSE N'Фиксированная'
    END AS rate_type,

    c.out_rub_close AS out_rub,

    CAST(
        c.int_num / NULLIF(c.int_den, 0)
        AS DECIMAL(12,6)
    ) AS rate_int_srvz,

    CAST(
        c.trf_num / NULLIF(c.trf_den, 0)
        AS DECIMAL(12,6)
    ) AS rate_trf_srvz

FROM #grid g

LEFT JOIN #closings c
    ON c.is_floatrate = g.is_floatrate
   AND c.d = g.d

ORDER BY
    g.d,
    g.is_floatrate;


    
