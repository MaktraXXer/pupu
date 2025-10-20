/* === ПАРАМЕТРЫ === */
DECLARE @dt_rep       date         = '2025-10-18';
DECLARE @cur          varchar(3)   = '810';
DECLARE @section_name nvarchar(50) = N'Срочные';
DECLARE @block_name   nvarchar(100)= N'Привлечение ФЛ';
DECLARE @acc_role     nvarchar(10) = N'LIAB';
DECLARE @od_only      bit          = 1;

DECLARE @eps          decimal(9,6) = 0.0005;
DECLARE @rk_split_from date        = '2025-10-16';

DECLARE @month_start date = DATEFROMPARTS(YEAR(@dt_rep), MONTH(@dt_rep), 1);

/* === РК-ставки === */
IF OBJECT_ID('tempdb..#rk_rates') IS NOT NULL DROP TABLE #rk_rates;
CREATE TABLE #rk_rates(d_from date,d_to date,conv_type varchar(20),r decimal(9,6));
INSERT INTO #rk_rates VALUES
(@month_start, DATEADD(day,-1,@rk_split_from),'AT_THE_END',0.165),
(@month_start, DATEADD(day,-1,@rk_split_from),'AT_THE_END',0.163),
(@month_start, DATEADD(day,-1,@rk_split_from),'NOT_AT_THE_END',0.162),
(@month_start, DATEADD(day,-1,@rk_split_from),'NOT_AT_THE_END',0.160),
(@rk_split_from,@dt_rep,'AT_THE_END',0.170),
(@rk_split_from,@dt_rep,'AT_THE_END',0.168),
(@rk_split_from,@dt_rep,'NOT_AT_THE_END',0.167),
(@rk_split_from,@dt_rep,'NOT_AT_THE_END',0.165);

/* === БАЗА: все депозиты (без фильтра по сроку) === */
WITH base_all AS (
    SELECT
        CAST(t.dt_open AS date) AS dt_open_d,
        t.con_id,
        t.cli_id,
        t.out_rub,
        t.rate_con,
        t.conv,
        t.termdays,
        t.prod_name_res
    FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
    WHERE
        t.dt_rep       = @dt_rep
        AND t.section_name = @section_name
        AND t.block_name   = @block_name
        AND (@od_only = 0 OR t.od_flag = 1)
        AND t.cur          = @cur
        AND t.acc_role     = @acc_role
        AND t.out_rub IS NOT NULL AND t.out_rub >= 0
        AND t.dt_open BETWEEN @month_start AND @dt_rep
),
/* === con_id из РК-91 (как в Скрипте 1) === */
rk91_base AS (
    SELECT *
    FROM base_all
    WHERE termdays BETWEEN 80 AND 100
),
rk91_by_con AS (
    SELECT
        CAST(dt_open AS date) AS dt_open_d,
        con_id,
        MIN(cli_id)   AS cli_id,
        SUM(out_rub)  AS out_rub,
        MIN(rate_con) AS rate_con,
        CASE
            WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(conv,''))),'')) IS NULL
                THEN 'AT_THE_END'
            ELSE UPPER(LTRIM(RTRIM(MIN(conv))))
        END AS conv_norm
    FROM rk91_base
    GROUP BY CAST(dt_open AS date), con_id
),
rk91_flag AS (
    SELECT
        c.*,
        CASE
            WHEN c.conv_norm = 'AT_THE_END' AND EXISTS (
                   SELECT 1 FROM #rk_rates rr
                   WHERE rr.conv_type='AT_THE_END'
                     AND c.dt_open_d BETWEEN rr.d_from AND rr.d_to
                     AND ABS(c.rate_con - rr.r) <= @eps
                 ) THEN 1
            WHEN c.conv_norm <> 'AT_THE_END' AND EXISTS (
                   SELECT 1 FROM #rk_rates rr
                   WHERE rr.conv_type='NOT_AT_THE_END'
                     AND c.dt_open_d BETWEEN rr.d_from AND rr.d_to
                     AND ABS(c.rate_con - rr.r) <= @eps
                 ) THEN 1
            ELSE 0
        END AS is_rk
    FROM rk91_by_con c
),
rk_ids AS (
    SELECT DISTINCT con_id
    FROM rk91_flag
    WHERE is_rk = 1
),
/* === МАРКЕТЫ, исключив РК-91 con_id === */
market_only AS (
    SELECT b.*
    FROM base_all b
    WHERE EXISTS (
        SELECT 1
        FROM (VALUES
            (N'Надёжный прайм'), (N'Надёжный VIP'), (N'Надёжный премиум'),
            (N'Надёжный промо'), (N'Надёжный старт'),
            (N'Надёжный Т2'),   (N'Надёжный T2'),
            (N'Надёжный Мегафон'), (N'Надёжный процент'),
            (N'Надёжный'), (N'ДОМа надёжно'), (N'Всё в ДОМ')
        ) m(name)
        WHERE m.name = b.prod_name_res
    )
      AND NOT EXISTS (SELECT 1 FROM rk_ids r WHERE r.con_id = b.con_id)
),
by_con AS (
    SELECT
        dt_open_d,
        con_id,
        MIN(cli_id)  AS cli_id,
        SUM(out_rub) AS out_rub
    FROM market_only
    GROUP BY dt_open_d, con_id
),
/* === РЕКУРСИВНЫЙ КАЛЕНДАРЬ === */
cal AS (
    SELECT @month_start AS open_date
    UNION ALL
    SELECT DATEADD(DAY, 1, open_date)
    FROM cal
    WHERE open_date < @dt_rep
)
SELECT
    c.open_date,
    COUNT(DISTINCT b.con_id)                  AS cnt_deposits_cum,
    COUNT(DISTINCT b.cli_id)                  AS cnt_cli_cum,
    SUM(b.out_rub)                            AS sum_out_rub_cum,
    CAST(SUM(b.out_rub)/1e9 AS decimal(18,6)) AS sum_out_rub_cum_bln
FROM cal c
LEFT JOIN by_con b
       ON b.dt_open_d <= c.open_date
GROUP BY c.open_date
ORDER BY c.open_date
OPTION (MAXRECURSION 32767);
