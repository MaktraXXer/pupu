
;WITH days AS (
    SELECT day_num
    FROM (VALUES
        (1), (4), (7), (10), (13),
        (16), (19), (22), (25), (28),
        (30), (32), (33), (36), (38)
    ) v(day_num)
),
dates AS (
    SELECT
        p.period_name,
        d.day_num,
        DATEADD(day, d.day_num - 1, p.dt_from) AS dt_rep
    FROM #periods p
    CROSS JOIN days d
    WHERE DATEADD(day, d.day_num - 1, p.dt_from) <= p.dt_to
),
balances AS (
    SELECT
        d.period_name,
        d.day_num,
        d.dt_rep,
        SUM(CASE WHEN e.category = N'НОВ'
                 THEN e.delta ELSE 0 END) AS nov,
        SUM(CASE WHEN e.category = N'НДП / НДМ'
                 THEN e.delta ELSE 0 END) AS ndm_ndp
    FROM dates d
    LEFT JOIN #events e
        ON e.period_name = d.period_name
       AND e.dt_rep <= d.dt_rep
    GROUP BY
        d.period_name,
        d.day_num,
        d.dt_rep
)
SELECT
    d.day_num AS [День РК],

    MAX(CASE WHEN b.period_name = N'Октябрь 2026'
        THEN CONVERT(varchar(10), b.dt_rep, 104)
    END) AS [Окт26 Дата],

    MAX(CASE WHEN b.period_name = N'Октябрь 2026'
        THEN CAST(b.nov / 1e9 AS decimal(18,6))
    END) AS [Окт26 НОВ],

    MAX(CASE WHEN b.period_name = N'Октябрь 2026'
        THEN CAST(b.ndm_ndp / 1e9 AS decimal(18,6))
    END) AS [Окт26 НДП/НДМ],

    MAX(CASE WHEN b.period_name = N'Март 2026'
        THEN CONVERT(varchar(10), b.dt_rep, 104)
    END) AS [Мар26 Дата],

    MAX(CASE WHEN b.period_name = N'Март 2026'
        THEN CAST(b.nov / 1e9 AS decimal(18,6))
    END) AS [Мар26 НОВ],

    MAX(CASE WHEN b.period_name = N'Март 2026'
        THEN CAST(b.ndm_ndp / 1e9 AS decimal(18,6))
    END) AS [Мар26 НДП/НДМ],

    MAX(CASE WHEN b.period_name = N'Октябрь 2025'
        THEN CONVERT(varchar(10), b.dt_rep, 104)
    END) AS [Окт25 Дата],

    MAX(CASE WHEN b.period_name = N'Октябрь 2025'
        THEN CAST(b.nov / 1e9 AS decimal(18,6))
    END) AS [Окт25 НОВ],

    MAX(CASE WHEN b.period_name = N'Октябрь 2025'
        THEN CAST(b.ndm_ndp / 1e9 AS decimal(18,6))
    END) AS [Окт25 НДП/НДМ]

FROM days d
LEFT JOIN balances b
    ON b.day_num = d.day_num
GROUP BY d.day_num
ORDER BY d.day_num;
