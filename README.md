
;WITH days AS (
    SELECT day_num
    FROM (VALUES
        (1), (4), (7), (10), (13),
        (16), (19), (22), (25), (28),
        (32), (33), (35), (36), (38)
    ) v(day_num)
),
balances AS (
    SELECT
        d.period_id,
        DATEDIFF(day, p.dt_from, d.dt_rep) + 1 AS day_num,
        CAST(
            (d.balance_deposits + d.balance_ns) / 1000000000.0
            AS decimal(18,6)
        ) AS total_bn
    FROM #daily d
    INNER JOIN #periods p
        ON p.period_id = d.period_id
)
SELECT
    CONVERT(
        varchar(10),
        DATEADD(day, d.day_num - 1, '2025-10-01'),
        104
    ) AS [Column1],

    MAX(CASE WHEN b.period_id = 1
        THEN b.total_bn END) AS [РК октябрь 26],

    MAX(CASE WHEN b.period_id = 2
        THEN b.total_bn END) AS [РК март 26],

    MAX(CASE WHEN b.period_id = 3
        THEN b.total_bn END) AS [РК октябрь 25]

FROM days d
LEFT JOIN balances b
    ON b.day_num = d.day_num

GROUP BY d.day_num
ORDER BY d.day_num;
