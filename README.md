, seq AS (
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub,
           win_start, win_end,
           cycle_id = CAST(cycle_id AS varchar(100))
    FROM seed
    UNION ALL
    SELECT s.con_id, s.cli_id, s.TSEGMENTNAME, s.out_rub,
           win_start = DATEADD(day, 1, s.win_end),
           win_end   = DATEADD(day, -1, EOMONTH(DATEADD(month, 1, DATEADD(day,1,s.win_end)))),
           cycle_id  = CAST(s.con_id AS varchar(20)) + '_' +
                       CAST(CAST(RIGHT(s.cycle_id, LEN(s.cycle_id) - CHARINDEX('_', s.cycle_id)) AS int) + 1 AS varchar(10))
    FROM   seq s
    WHERE  DATEADD(day, 1, s.win_end) <= @HorizonTo
)
