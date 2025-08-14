/* --- Параметры --- */
DECLARE @JulEOM     date = '2025-07-31';
DECLARE @ChkDate    date = '2025-08-12';
DECLARE @CloseFrom  date = '2025-08-01';
DECLARE @CloseTo    date = '2025-08-11';

/* --- ФУ продукты --- */
DECLARE @FU TABLE (prod_name_res nvarchar(255) PRIMARY KEY);
INSERT INTO @FU(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* --- Источник --- */
WITH base AS (
    SELECT
        t.dt_rep,
        t.cli_id,
        t.con_id,
        t.PROD_NAME_res,
        CAST(t.out_rub AS decimal(20,2)) AS out_rub,
        CAST(t.dt_close AS date)         AS dt_close
    FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
    WHERE t.block_name = N'Привлечение ФЛ'
      AND t.od_flag    = 1
      AND t.cur        = '810'
      AND t.out_rub   IS NOT NULL
      AND t.section_name IN (N'Срочные', N'Срочные ')
      AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
),
/* --- Когорта --- */
cohort AS (
    SELECT DISTINCT b.cli_id
    FROM base b
    JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_close BETWEEN @CloseFrom AND @CloseTo
),
/* --- Срезы --- */
snap_0731 AS (
    SELECT
        b.cli_id,
        SUM(CASE WHEN b.out_rub < 0 THEN 0 ELSE b.out_rub END) AS vol_total_0731,
        MAX(CASE WHEN f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END) AS has_fu_0731,
        MAX(CASE WHEN f.prod_name_res IS NULL THEN 1 ELSE 0 END)     AS has_nonfu_0731
    FROM base b
    JOIN cohort c ON c.cli_id = b.cli_id
    LEFT JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_rep = @JulEOM
    GROUP BY b.cli_id
),
snap_0812 AS (
    SELECT
        b.cli_id,
        SUM(CASE WHEN b.out_rub < 0 THEN 0 ELSE b.out_rub END) AS vol_total_0812,
        MAX(CASE WHEN f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END) AS has_fu_0812,
        MAX(CASE WHEN f.prod_name_res IS NULL THEN 1 ELSE 0 END)     AS has_nonfu_0812
    FROM base b
    JOIN cohort c ON c.cli_id = b.cli_id
    LEFT JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_rep = @ChkDate
    GROUP BY b.cli_id
),
/* --- Деталка с классификацией --- */
detail AS (
    SELECT
        c.cli_id,
        ISNULL(s7.vol_total_0731,0) AS vol_total_0731,
        ISNULL(s8.vol_total_0812,0) AS vol_total_0812,

        CASE 
          WHEN ISNULL(s7.has_fu_0731,0)=1 AND ISNULL(s7.has_nonfu_0731,0)=0 THEN 'A0'
          WHEN ISNULL(s7.has_fu_0731,0)=1 AND ISNULL(s7.has_nonfu_0731,0)=1 THEN 'B0'
          ELSE 'OUT'
        END AS initial_state,

        CASE 
          WHEN ISNULL(s8.has_fu_0812,0)=1 AND ISNULL(s8.has_nonfu_0812,0)=0 THEN 'A1'
          WHEN ISNULL(s8.has_fu_0812,0)=1 AND ISNULL(s8.has_nonfu_0812,0)=1 THEN 'B1'
          WHEN ISNULL(s8.has_fu_0812,0)=0 AND ISNULL(s8.has_nonfu_0812,0)=1 THEN 'C1'
          ELSE 'N1'
        END AS final_state
    FROM cohort c
    LEFT JOIN snap_0731 s7 ON s7.cli_id = c.cli_id
    LEFT JOIN snap_0812 s8 ON s8.cli_id = c.cli_id
)
/* --- Итог 8 путей --- */
SELECT
    initial_state,
    final_state,
    SUM(vol_total_0731)                         AS vol_total_start,
    COUNT(DISTINCT cli_id)                      AS clients_start,
    COUNT(DISTINCT CASE WHEN vol_total_0812>0 THEN cli_id END) AS clients_end,
    SUM(vol_total_0812)                         AS vol_total_end
FROM detail
WHERE initial_state IN ('A0','B0')
GROUP BY initial_state, final_state
ORDER BY initial_state, final_state;
