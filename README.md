DECLARE @dt_start DATE = '2025-10-01';
DECLARE @dt_end   DATE = '2025-10-31';  -- inclusive
DECLARE @threshold_rub BIGINT = 3000000000; -- 3 млрд

;WITH base AS (
    SELECT
        b.dt_rep,
        b.cli_id,
        SUM(b.out_rub) AS sum_rub
    FROM [ALM].[ALM].[balance_rest_all] b WITH (NOLOCK)
    WHERE b.dt_rep >= @dt_start
      AND b.dt_rep <= @dt_end
      AND b.cur = '810'
      AND b.section_name IN (N'До востребования', N'Накопительный счёт')
      AND b.od_flag = 1
      AND b.block_name = N'Привлечение ФЛ'
    GROUP BY b.dt_rep, b.cli_id
),
qualified AS (
    SELECT
        dt_rep,
        cli_id,
        sum_rub
    FROM base
    WHERE sum_rub >= @threshold_rub
),
agg AS (
    SELECT
        dt_rep,
        SUM(sum_rub) AS out_rub,
        STRING_AGG(CONVERT(NVARCHAR(30), cli_id), N';') AS multiple_cli_id
    FROM qualified
    GROUP BY dt_rep
),
dates AS (
    SELECT @dt_start AS dt_rep
    UNION ALL
    SELECT DATEADD(DAY, 1, dt_rep)
    FROM dates
    WHERE dt_rep < @dt_end
)
SELECT
    d.dt_rep,
    COALESCE(a.out_rub, 0) AS out_rub,
    a.multiple_cli_id
FROM dates d
LEFT JOIN agg a
    ON a.dt_rep = d.dt_rep
ORDER BY d.dt_rep
OPTION (MAXRECURSION 32767);


/* -------------------------------
   СИНТЕТИЧЕСКИЙ ТЕСТ (без записи в ALM)
   ------------------------------- */

DECLARE @dt_start DATE = '2026-01-01';
DECLARE @dt_end   DATE = '2026-01-20';  -- inclusive
DECLARE @threshold_rub BIGINT = 3000000000; -- 3 млрд

IF OBJECT_ID('tempdb..#balance_rest_all_test') IS NOT NULL
    DROP TABLE #balance_rest_all_test;

CREATE TABLE #balance_rest_all_test (
    dt_rep       DATE           NOT NULL,
    cli_id       INT            NOT NULL,
    out_rub      BIGINT         NOT NULL,
    cur          NVARCHAR(3)    NOT NULL,
    section_name NVARCHAR(100)  NOT NULL,
    od_flag      BIT            NOT NULL,
    block_name   NVARCHAR(100)  NOT NULL
);

;WITH d AS (
    SELECT CAST('2026-01-01' AS DATE) AS dt_rep
    UNION ALL
    SELECT DATEADD(DAY, 1, dt_rep)
    FROM d
    WHERE dt_rep < '2026-01-20'
),
ins AS (
    -- Клиент 666: 4 млрд с 01 по 15 января 2026
    SELECT dt_rep, 666 AS cli_id, CAST(4000000000 AS BIGINT) AS out_rub,
           N'810' AS cur, N'Накопительный счёт' AS section_name,
           CAST(1 AS BIT) AS od_flag, N'Привлечение ФЛ' AS block_name
    FROM d
    WHERE dt_rep BETWEEN '2026-01-01' AND '2026-01-15'

    UNION ALL

    -- Клиент 444: 50 млрд с 10 по 16 января 2026
    SELECT dt_rep, 444 AS cli_id, CAST(50000000000 AS BIGINT) AS out_rub,
           N'810' AS cur, N'До востребования' AS section_name,
           CAST(1 AS BIT) AS od_flag, N'Привлечение ФЛ' AS block_name
    FROM d
    WHERE dt_rep BETWEEN '2026-01-10' AND '2026-01-16'
)
INSERT INTO #balance_rest_all_test (dt_rep, cli_id, out_rub, cur, section_name, od_flag, block_name)
SELECT dt_rep, cli_id, out_rub, cur, section_name, od_flag, block_name
FROM ins
OPTION (MAXRECURSION 32767);


/* -------------------------------
   ОСНОВНОЙ ЗАПРОС (как в проде, но на #таблице)
   ------------------------------- */

;WITH base AS (
    SELECT
        b.dt_rep,
        b.cli_id,
        SUM(b.out_rub) AS sum_rub
    FROM #balance_rest_all_test b
    WHERE b.dt_rep >= @dt_start
      AND b.dt_rep <= @dt_end
      AND b.cur = N'810'
      AND b.section_name IN (N'До востребования', N'Накопительный счёт')
      AND b.od_flag = 1
      AND b.block_name = N'Привлечение ФЛ'
    GROUP BY b.dt_rep, b.cli_id
),
qualified AS (
    SELECT dt_rep, cli_id, sum_rub
    FROM base
    WHERE sum_rub >= @threshold_rub
),
agg AS (
    SELECT
        dt_rep,
        SUM(sum_rub) AS out_rub,
        STRING_AGG(CONVERT(NVARCHAR(30), cli_id), N';') AS multiple_cli_id
    FROM qualified
    GROUP BY dt_rep
),
dates AS (
    SELECT @dt_start AS dt_rep
    UNION ALL
    SELECT DATEADD(DAY, 1, dt_rep)
    FROM dates
    WHERE dt_rep < @dt_end
)
SELECT
    d.dt_rep,
    COALESCE(a.out_rub, 0) AS out_rub,
    a.multiple_cli_id
FROM dates d
LEFT JOIN agg a
    ON a.dt_rep = d.dt_rep
ORDER BY d.dt_rep
OPTION (MAXRECURSION 32767);
