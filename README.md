/*=====================================================================
  П А Р А М Е Т Р Ы
=====================================================================*/
DECLARE @rep_date  date = '2025-05-11';   -- последняя дата окна
DECLARE @days_back int  = 6;              -- 6 дней назад + сама дата = 7 суток

/*=====================================================================
  1. Суточные остатки клиента по трём секциям
=====================================================================*/
;WITH daily_cli AS (
    SELECT
        db.dt_rep,
        db.cli_id,

        /* ДВС --------------------------------------------------------*/
        SUM(CASE
                WHEN LTRIM(RTRIM(db.section_name)) = N'До востребования'
                     THEN db.out_rub
            END) AS dv_rub,

        /* НС ---------------------------------------------------------*/
        SUM(CASE
                WHEN LTRIM(RTRIM(db.section_name)) = N'Накопительный счёт'
                     THEN db.out_rub
            END) AS ns_rub,

        /* Срочные ----------------------------------------------------*/
        SUM(CASE
                WHEN LTRIM(RTRIM(db.section_name)) = N'Срочные'
                     THEN db.out_rub
            END) AS sr_rub
    FROM ALM.balance_rest_all db WITH (NOLOCK)
    WHERE db.dt_rep BETWEEN DATEADD(day,-@days_back,@rep_date) AND @rep_date
      AND db.CUR          = '810'                     -- RUB
      AND db.MAP_IS_CASH  = 1
      AND db.TSEGMENTNAME = N'Розничный бизнес'
      AND db.AP           = N'Пассив'
      AND db.BLOCK_NAME   = N'Привлечение ФЛ'
      AND LTRIM(RTRIM(db.section_name)) IN
            (N'Срочные', N'До востребования', N'Накопительный счёт')
    GROUP BY db.dt_rep, db.cli_id
),

/*=====================================================================
  2. Средний ДВС клиента за неделю  →  бакет
=====================================================================*/
weekly_avg AS (
    SELECT
        dc.cli_id,
        AVG(ISNULL(dc.dv_rub,0)) AS avg_dv
    FROM daily_cli dc
    GROUP BY dc.cli_id
),
bucket_map AS (
    SELECT
        wa.cli_id,
        CASE
            WHEN wa.avg_dv IS NULL OR wa.avg_dv <=      0     THEN N'0 / null'
            WHEN wa.avg_dv <        5000                     THEN N'< 5 тыс'
            WHEN wa.avg_dv <       10000                     THEN N'< 10 тыс'
            WHEN wa.avg_dv <       50000                     THEN N'< 50 тыс'
            WHEN wa.avg_dv <      150000                     THEN N'< 150 тыс'
            WHEN wa.avg_dv <      500000                     THEN N'< 500 тыс'
            ELSE                                                N'≥ 500 тыс'
        END AS bucket
    FROM weekly_avg wa
),

/*=====================================================================
  3. Суточные остатки + бакет клиента
=====================================================================*/
daily_with_bucket AS (
    SELECT
        dc.dt_rep,
        bm.bucket,
        dc.cli_id,
        ISNULL(dc.dv_rub,0) AS dv_rub,
        ISNULL(dc.ns_rub,0) AS ns_rub,
        ISNULL(dc.sr_rub,0) AS sr_rub
    FROM daily_cli dc
    JOIN bucket_map bm ON bm.cli_id = dc.cli_id
)

/*=====================================================================
  4. Итог по бакету и дате (6 столбцов)
=====================================================================*/
SELECT
    d.dt_rep,
    d.bucket,

    /* --- Накопительный счёт ---------------------------------------*/
    COUNT(DISTINCT CASE WHEN d.ns_rub > 0 THEN d.cli_id END)                      AS cnt_ns_cli,
    CASE
        WHEN COUNT(DISTINCT CASE WHEN d.ns_rub > 0 THEN d.cli_id END) = 0
             THEN 0
        ELSE
             SUM(CASE WHEN d.ns_rub > 0 THEN d.ns_rub END) /
             COUNT(DISTINCT CASE WHEN d.ns_rub > 0 THEN d.cli_id END)
    END                                                                            AS avg_ns_vol,

    /* --- До востребования -----------------------------------------*/
    COUNT(DISTINCT CASE WHEN d.dv_rub > 0 THEN d.cli_id END)                      AS cnt_dv_cli,
    CASE
        WHEN COUNT(DISTINCT CASE WHEN d.dv_rub > 0 THEN d.cli_id END) = 0
             THEN 0
        ELSE
             SUM(CASE WHEN d.dv_rub > 0 THEN d.dv_rub END) /
             COUNT(DISTINCT CASE WHEN d.dv_rub > 0 THEN d.cli_id END)
    END                                                                            AS avg_dv_vol,

    /* --- Срочные --------------------------------------------------*/
    COUNT(DISTINCT CASE WHEN d.sr_rub > 0 THEN d.cli_id END)                      AS cnt_sr_cli,
    CASE
        WHEN COUNT(DISTINCT CASE WHEN d.sr_rub > 0 THEN d.cli_id END) = 0
             THEN 0
        ELSE
             SUM(CASE WHEN d.sr_rub > 0 THEN d.sr_rub END) /
             COUNT(DISTINCT CASE WHEN d.sr_rub > 0 THEN d.cli_id END)
    END                                                                            AS avg_sr_vol
FROM daily_with_bucket d
GROUP BY
    d.dt_rep,
    d.bucket
ORDER BY
    d.dt_rep,
    CASE d.bucket
        WHEN N'0 / null'  THEN 0
        WHEN N'< 5 тыс'   THEN 1
        WHEN N'< 10 тыс'  THEN 2
        WHEN N'< 50 тыс'  THEN 3
        WHEN N'< 150 тыс' THEN 4
        WHEN N'< 500 тыс' THEN 5
        ELSE                    6
    END;
