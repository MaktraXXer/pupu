/* ======================= ПАРАМЕТРЫ ======================= */
DECLARE @JulEOM     date = '2025-07-31';
DECLARE @ChkDate    date = '2025-08-12';
DECLARE @CloseFrom  date = '2025-08-01';
DECLARE @CloseTo    date = '2025-08-11';

/* Справочник ФУ-продуктов */
DECLARE @FU TABLE (prod_name_res nvarchar(255) PRIMARY KEY);
INSERT INTO @FU(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* ======================= ИСТОЧНИК + КОГОРТА ======================= */
WITH base AS (
    SELECT
        t.dt_rep,
        t.cli_id,
        t.con_id,
        t.section_name,
        t.PROD_NAME_res,
        t.TSEGMENTNAME,
        CAST(t.out_rub AS decimal(20,2))        AS out_rub,
        CAST(t.dt_open AS date)                  AS dt_open,
        CAST(t.dt_close AS date)                 AS dt_close
    FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
    WHERE t.block_name = N'Привлечение ФЛ'
      AND t.od_flag    = 1
      AND t.cur        = '810'
      AND t.out_rub   IS NOT NULL
      AND t.section_name IN (N'Срочные', N'Срочные ')
      AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
),
cohort AS (  -- клиенты, у кого закрытия ФУ попали в окно 01–11.08
    SELECT DISTINCT b.cli_id
    FROM base b
    JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_close BETWEEN @CloseFrom AND @CloseTo
),
/* ======================= СРЕЗЫ НА 31.07 И 12.08 ======================= */
snap_0731 AS (
    SELECT
        b.cli_id,
        SUM(CASE WHEN f.prod_name_res IS NOT NULL
                 THEN CASE WHEN b.out_rub < 0 THEN 0 ELSE b.out_rub END ELSE 0 END) AS vol_fu_0731,
        SUM(CASE WHEN f.prod_name_res IS NULL
                 THEN CASE WHEN b.out_rub < 0 THEN 0 ELSE b.out_rub END ELSE 0 END) AS vol_nonfu_0731,
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
        SUM(CASE WHEN f.prod_name_res IS NOT NULL
                 THEN CASE WHEN b.out_rub < 0 THEN 0 ELSE b.out_rub END ELSE 0 END) AS vol_fu_0812,
        SUM(CASE WHEN f.prod_name_res IS NULL
                 THEN CASE WHEN b.out_rub < 0 THEN 0 ELSE b.out_rub END ELSE 0 END) AS vol_nonfu_0812,
        MAX(CASE WHEN f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END) AS has_fu_0812,
        MAX(CASE WHEN f.prod_name_res IS NULL THEN 1 ELSE 0 END)     AS has_nonfu_0812
    FROM base b
    JOIN cohort c ON c.cli_id = b.cli_id
    LEFT JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_rep = @ChkDate
    GROUP BY b.cli_id
),
/* (опц.) открытия в окне 01–12.08 — для дополнительной аналитики */
opened_fu_win AS (
    SELECT DISTINCT b.cli_id
    FROM base b
    JOIN cohort c ON c.cli_id = b.cli_id
    JOIN @FU f     ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_open BETWEEN @CloseFrom AND @ChkDate
),
opened_nonfu_win AS (
    SELECT DISTINCT b.cli_id
    FROM base b
    JOIN cohort c ON c.cli_id = b.cli_id
    LEFT JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE f.prod_name_res IS NULL
      AND b.dt_open BETWEEN @CloseFrom AND @ChkDate
),
/* ======================= ДЕТАЛИ ПО КЛИЕНТАМ + КЛАССИФИКАЦИЯ ======================= */
detail AS (
    SELECT
        c.cli_id,

        /* объёмы/флаги на 31.07 */
        ISNULL(s7.vol_fu_0731,0)      AS vol_fu_0731,
        ISNULL(s7.vol_nonfu_0731,0)   AS vol_nonfu_0731,
        ISNULL(s7.has_fu_0731,0)      AS has_fu_0731,
        ISNULL(s7.has_nonfu_0731,0)   AS has_nonfu_0731,

        /* объёмы/флаги на 12.08 */
        ISNULL(s8.vol_fu_0812,0)      AS vol_fu_0812,
        ISNULL(s8.vol_nonfu_0812,0)   AS vol_nonfu_0812,
        ISNULL(s8.has_fu_0812,0)      AS has_fu_0812,
        ISNULL(s8.has_nonfu_0812,0)   AS has_nonfu_0812,

        /* маркеры открытий в окне 01–12.08 */
        CASE WHEN ofu.cli_id    IS NULL THEN 0 ELSE 1 END AS opened_fu_win,
        CASE WHEN onf.cli_id    IS NULL THEN 0 ELSE 1 END AS opened_nonfu_win,

        /* исходное состояние на 31.07: A0 / B0 (прочие исключаем из матрицы) */
        CASE 
          WHEN ISNULL(s7.has_fu_0731,0)=1 AND ISNULL(s7.has_nonfu_0731,0)=0 THEN 'A0'
          WHEN ISNULL(s7.has_fu_0731,0)=1 AND ISNULL(s7.has_nonfu_0731,0)=1 THEN 'B0'
          ELSE 'OUT' 
        END AS initial_state,

        /* целевое состояние на 12.08: A1/B1/C1/N1 */
        CASE 
          WHEN ISNULL(s8.has_fu_0812,0)=1 AND ISNULL(s8.has_nonfu_0812,0)=0 THEN 'A1'
          WHEN ISNULL(s8.has_fu_0812,0)=1 AND ISNULL(s8.has_nonfu_0812,0)=1 THEN 'B1'
          WHEN ISNULL(s8.has_fu_0812,0)=0 AND ISNULL(s8.has_nonfu_0812,0)=1 THEN 'C1'
          ELSE 'N1'
        END AS final_state
    FROM cohort c
    LEFT JOIN snap_0731 s7 ON s7.cli_id = c.cli_id
    LEFT JOIN snap_0812 s8 ON s8.cli_id = c.cli_id
    LEFT JOIN opened_fu_win     ofu ON ofu.cli_id = c.cli_id
    LEFT JOIN opened_nonfu_win  onf ON onf.cli_id = c.cli_id
)
/* ======================= ИТОГОВАЯ ТАБЛИЦА 8 ПУТЕЙ ======================= */
SELECT
    d.initial_state,                       -- 'A0' / 'B0'
    d.final_state,                         -- 'A1' / 'B1' / 'C1' / 'N1'
    COUNT(DISTINCT d.cli_id) AS clients_cnt,

    /* объёмы на 31.07 */
    SUM(d.vol_fu_0731)      AS vol_fu_0731,
    SUM(d.vol_nonfu_0731)   AS vol_nonfu_0731,
    SUM(d.vol_fu_0731 + d.vol_nonfu_0731) AS vol_total_0731,

    /* объёмы на 12.08 */
    SUM(d.vol_fu_0812)      AS vol_fu_0812,
    SUM(d.vol_nonfu_0812)   AS vol_nonfu_0812,
    SUM(d.vol_fu_0812 + d.vol_nonfu_0812) AS vol_total_0812,

    /* (опц.) метка «переоткрылся только на ФУ» в окне 01–12.08 */
    SUM(CASE WHEN d.opened_fu_win=1 AND d.opened_nonfu_win=0 THEN 1 ELSE 0 END) AS reopened_only_fu_cnt,

    /* доля цепочки внутри своей исходной группы (по клиентам) — для удобства оценки */
    CAST(100.0 * COUNT(*) 
         / NULLIF(SUM(COUNT(*)) OVER (PARTITION BY d.initial_state),0) AS decimal(5,2)) AS pct_within_initial_grp
FROM detail d
WHERE d.initial_state IN ('A0','B0')   -- только 2 стартовые группы
GROUP BY d.initial_state, d.final_state
ORDER BY d.initial_state, d.final_state;
