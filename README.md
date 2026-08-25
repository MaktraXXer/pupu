DECLARE @dt_start DATE = '2026-03-12';
DECLARE @dt_end   DATE = '2026-03-31';

;WITH base AS (
    SELECT
        b.dt_rep,
        b.cli_id,
        SUM(b.out_rub) AS client_out_rub
    FROM [ALM].[ALM].[VW_Balance_Rest_All] b WITH (NOLOCK)
    WHERE b.dt_rep >= @dt_start
      AND b.dt_rep <= @dt_end
      AND b.cur = '810'
      AND b.section_name = N'Накопительный счёт'
      AND b.od_flag = 1
      AND b.prod_id = 654
      AND b.block_name = N'Привлечение ФЛ'
    GROUP BY
        b.dt_rep,
        b.cli_id
),
agg AS (
    SELECT
        dt_rep,

        SUM(
            CASE
                WHEN client_out_rub <= 1500000
                THEN client_out_rub
                ELSE 0
            END
        ) AS out_rub_to_1_5m,

        SUM(
            CASE
                WHEN client_out_rub > 1500000
                 AND client_out_rub <= 10000000
                THEN client_out_rub
                ELSE 0
            END
        ) AS out_rub_1_5m_to_10m,

        SUM(
            CASE
                WHEN client_out_rub > 10000000
                 AND client_out_rub <= 30000000
                THEN client_out_rub
                ELSE 0
            END
        ) AS out_rub_10m_to_30m,

        SUM(
            CASE
                WHEN client_out_rub > 30000000
                THEN client_out_rub
                ELSE 0
            END
        ) AS out_rub_over_30m

    FROM base
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

    COALESCE(a.out_rub_to_1_5m, 0)
        AS [Остаток до 1.5 млн включительно],

    COALESCE(a.out_rub_1_5m_to_10m, 0)
        AS [Остаток свыше 1.5 до 10 млн включительно],

    COALESCE(a.out_rub_10m_to_30m, 0)
        AS [Остаток свыше 10 до 30 млн включительно],

    COALESCE(a.out_rub_over_30m, 0)
        AS [Остаток свыше 30 млн]

FROM dates d
LEFT JOIN agg a
    ON a.dt_rep = d.dt_rep
ORDER BY d.dt_rep
OPTION (MAXRECURSION 0);
