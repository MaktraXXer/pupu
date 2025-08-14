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
        t.section_name,
        CAST(t.out_rub AS decimal(20,2)) AS out_rub,
        CAST(t.dt_close AS date)         AS dt_close
    FROM alm.ALM.balance_rest_all t WITH (NOLOCK) -- без vw_ для скорости
    WHERE t.block_name = N'Привлечение ФЛ'
      AND t.od_flag    = 1
      AND t.cur        = '810'
      AND t.out_rub   IS NOT NULL
      AND (
            t.section_name IN (N'Срочные', N'Срочные ', N'Накопительный счёт')
          )
      AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
),
/* --- Когорта закрытий ФУ --- */
cohort AS (
    SELECT 
        b.cli_id,
        SUM(CASE WHEN b.out_rub < 0 THEN 0 ELSE b.out_rub END) AS vol_closed_fu
    FROM base b
    JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_close BETWEEN @CloseFrom AND @CloseTo
    GROUP BY b.cli_id
),
/* --- Срез на дату --- */
snap AS (
    SELECT
        b.dt_rep,
        b.cli_id,
        SUM(CASE WHEN f.prod_name_res IS NOT NULL THEN b.out_rub ELSE 0 END) AS vol_fu,
        SUM(CASE WHEN f.prod_name_res IS NULL AND b.section_name <> N'Накопительный счёт' THEN b.out_rub ELSE 0 END) AS vol_nonfu,
        SUM(CASE WHEN b.section_name = N'Накопительный счёт' THEN b.out_rub ELSE 0 END) AS vol_ns
    FROM base b
    JOIN cohort c ON c.cli_id = b.cli_id
    LEFT JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_rep IN (@JulEOM, @ChkDate)
    GROUP BY b.dt_rep, b.cli_id
),
/* --- Аггрегат по датам --- */
agg AS (
    SELECT
        s.dt_rep,
        SUM(s.vol_fu)    AS total_fu,
        SUM(s.vol_nonfu) AS total_nonfu,
        SUM(s.vol_ns)    AS total_ns,
        COUNT(DISTINCT s.cli_id) AS clients_cnt
    FROM snap s
    GROUP BY s.dt_rep
)
/* --- Итог --- */
SELECT
    (SELECT SUM(vol_closed_fu) FROM cohort) AS closed_fu_volume,
    (SELECT COUNT(*) FROM cohort)           AS closed_fu_clients,
    a.*
FROM agg a
ORDER BY a.dt_rep;
