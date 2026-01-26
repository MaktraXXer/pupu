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
)
SELECT
    d.dt_rep,
    COALESCE(a.out_rub, 0) AS out_rub,
    a.multiple_cli_id
FROM (
    -- генератор календаря дат в диапазоне
    SELECT DATEADD(DAY, v.number, @dt_start) AS dt_rep
    FROM master..spt_values v
    WHERE v.[type] = 'P'
      AND v.number BETWEEN 0 AND DATEDIFF(DAY, @dt_start, @dt_end)
) d
LEFT JOIN agg a
    ON a.dt_rep = d.dt_rep
ORDER BY d.dt_rep;
