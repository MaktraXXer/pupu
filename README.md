/* === Параметры ========================================================= */
DECLARE @rep_date  date = '2025-05-11';
DECLARE @days_back int  = 6;

/* === 1. Суточные остатки клиента ======================================= */
;WITH daily_cli AS (
    SELECT
        db.dt_rep,
        db.cli_id,
        SUM(CASE WHEN LTRIM(RTRIM(db.section_name)) = N'До востребования'
                 THEN db.out_rub END) AS dv_rub,
        SUM(CASE WHEN LTRIM(RTRIM(db.section_name)) = N'Накопительный счёт'
                 THEN db.out_rub END) AS ns_rub,
        SUM(CASE WHEN LTRIM(RTRIM(db.section_name)) = N'Срочные'
                 THEN db.out_rub END) AS sr_rub
    FROM ALM.balance_rest_all db WITH (NOLOCK)
    WHERE db.dt_rep BETWEEN DATEADD(day,-@days_back,@rep_date) AND @rep_date
      AND db.CUR          = '810'
      AND db.MAP_IS_CASH  = 1
      AND db.TSEGMENTNAME = N'Розничный бизнес'
      AND db.AP           = N'Пассив'
      AND db.BLOCK_NAME   = N'Привлечение ФЛ'
      AND LTRIM(RTRIM(db.section_name)) IN
          (N'Срочные', N'До востребования', N'Накопительный счёт')
    GROUP BY db.dt_rep, db.cli_id
),

/* === 2. Средний ДВС-остаток клиента за 7 суток → бакет ================== */
weekly_avg AS (
    SELECT cli_id,
           AVG(ISNULL(dv_rub,0)) AS avg_dv
    FROM   daily_cli
    GROUP  BY cli_id
),
bucket_map AS (
    SELECT
        wa.cli_id,
        CASE
            WHEN wa.avg_dv <=     0       THEN N'0 / null'
            WHEN wa.avg_dv <     5000     THEN N'< 5 тыс'
            WHEN wa.avg_dv <    10000     THEN N'< 10 тыс'
            WHEN wa.avg_dv <    50000     THEN N'< 50 тыс'
            WHEN wa.avg_dv <   150000     THEN N'< 150 тыс'
            WHEN wa.avg_dv <   500000     THEN N'< 500 тыс'
            ELSE                              N'≥ 500 тыс'
        END AS bucket
    FROM weekly_avg wa
),

/* === 3. Суточные остатки + бакет клиента ================================= */
daily_with_bucket AS (
    SELECT
        dc.dt_rep,
        bm.bucket,
        dc.cli_id,
        ISNULL(dc.dv_rub,0) AS dv_rub,
        ISNULL(dc.ns_rub,0) AS ns_rub,
        ISNULL(dc.sr_rub,0) AS sr_rub
    FROM  daily_cli dc
    JOIN  bucket_map bm ON bm.cli_id = dc.cli_id
),

/* === 4. Суммарный объём секции в бакете за дату ========================= */
bucket_vol AS (
    SELECT
        dt_rep,
        bucket,
        SUM(dv_rub) AS dv_vol,
        SUM(ns_rub) AS ns_vol,
        SUM(sr_rub) AS sr_vol
    FROM daily_with_bucket
    GROUP BY dt_rep, bucket
)

/* === 5. Финальный вывод: объёмы и доли ================================== */
SELECT
    bv.dt_rep                                    AS [Дата],
    bv.bucket                                    AS [Бакет avg(ДВС-7д)],

    /* -------- ДВС -------- */
    bv.dv_vol            / 1e6                           AS dv_vol_mln,
    bv.dv_vol * 1.0 /
        NULLIF( SUM(bv.dv_vol) OVER (PARTITION BY bv.dt_rep), 0 )   AS dv_share,

    /* -------- НС --------- */
    bv.ns_vol            / 1e6                           AS ns_vol_mln,
    bv.ns_vol * 1.0 /
        NULLIF( SUM(bv.ns_vol) OVER (PARTITION BY bv.dt_rep), 0 )   AS ns_share,

    /* -------- Срочные ---- */
    bv.sr_vol            / 1e6                           AS dep_vol_mln,
    bv.sr_vol * 1.0 /
        NULLIF( SUM(bv.sr_vol) OVER (PARTITION BY bv.dt_rep), 0 )   AS dep_share
FROM bucket_vol bv
ORDER BY
    bv.dt_rep,
    CASE bv.bucket
        WHEN N'0 / null'  THEN 0
        WHEN N'< 5 тыс'   THEN 1
        WHEN N'< 10 тыс'  THEN 2
        WHEN N'< 50 тыс'  THEN 3
        WHEN N'< 150 тыс' THEN 4
        WHEN N'< 500 тыс' THEN 5
        ELSE                    6
    END;
