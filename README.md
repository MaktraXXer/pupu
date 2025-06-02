/*========================================================================
  ПАРАМЕТРЫ
========================================================================*/
DECLARE @rep_date  date = '2025-05-11';   -- последняя дата окна
DECLARE @days_back int  = 6;              -- 6 дней назад + сама дата = 7 суток

/*========================================================================
  1. Остатки клиента по секциям на КАЖДУЮ дату
========================================================================*/
;WITH daily_cli AS (
    SELECT
        db.dt_rep,
        db.cli_id,

        SUM(CASE 
                WHEN LTRIM(RTRIM(db.section_name)) = N'До востребования'
                     THEN db.out_rub
            END) AS dv_rub,

        SUM(CASE 
                WHEN LTRIM(RTRIM(db.section_name)) = N'Накопительный счёт'
                     THEN db.out_rub
            END) AS ns_rub,

        SUM(CASE 
                WHEN LTRIM(RTRIM(db.section_name)) = N'Срочные'
                     THEN db.out_rub
            END) AS sr_rub
    FROM ALM.balance_rest_all db WITH (NOLOCK)
    WHERE db.dt_rep BETWEEN DATEADD(day, -@days_back, @rep_date) AND @rep_date
      AND db.CUR          = '810'                     -- RUB
      AND db.MAP_IS_CASH  = 1
      AND db.TSEGMENTNAME = N'Розничный бизнес'
      AND db.AP           = N'Пассив'
      AND db.BLOCK_NAME   = N'Привлечение ФЛ'
      AND LTRIM(RTRIM(db.section_name)) IN
            (N'Срочные', N'До востребования', N'Накопительный счёт')
    GROUP BY db.dt_rep, db.cli_id
),

/*========================================================================
  2. Присваиваем бакет по ДВС-остатку ТОГО дня
========================================================================*/
bucket_day AS (
    SELECT
        dc.dt_rep,
        dc.cli_id,
        dc.dv_rub,
        dc.ns_rub,
        dc.sr_rub,

        CASE
            WHEN dc.dv_rub IS NULL OR dc.dv_rub <= 0      THEN N'0 / null'
            WHEN dc.dv_rub <     5000                    THEN N'< 5 тыс'
            WHEN dc.dv_rub <    10000                    THEN N'< 10 тыс'
            WHEN dc.dv_rub <    50000                    THEN N'< 50 тыс'
            WHEN dc.dv_rub <   150000                    THEN N'< 150 тыс'
            WHEN dc.dv_rub <   500000                    THEN N'< 500 тыс'
            ELSE                                            N'≥ 500 тыс'
        END AS bucket
    FROM daily_cli dc
),

/*========================================================================
  3. Итог по бакету в каждую дату
========================================================================*/
bucket_tot AS (
    SELECT
        bd.dt_rep,
        bd.bucket,

        COUNT(DISTINCT bd.cli_id)            AS clients_day,
        SUM(ISNULL(bd.dv_rub,0))             AS dv_sum_day,
        SUM(ISNULL(bd.ns_rub,0))             AS ns_sum_day,
        SUM(ISNULL(bd.sr_rub,0))             AS sr_sum_day
    FROM bucket_day bd
    GROUP BY bd.dt_rep, bd.bucket
),

/*========================================================================
  4. Среднее по 7 датам для каждого бакета
========================================================================*/
bucket_avg AS (
    SELECT
        bt.bucket,

        AVG(bt.clients_day)  AS avg_clients,
        AVG(bt.dv_sum_day)   AS avg_dv,
        AVG(bt.ns_sum_day)   AS avg_ns,
        AVG(bt.sr_sum_day)   AS avg_sr
    FROM bucket_tot bt
    GROUP BY bt.bucket
)

/*========================================================================
  5. Финальный вывод (суммы – в млн ₽)
========================================================================*/
SELECT
    ba.bucket                                         AS [Бакет (ДВС-на-дату)],

    CAST(ba.avg_clients AS decimal(18,2))             AS avg_clients_per_day,

    ba.avg_dv / 1e6              AS dv_avg_7d_mln,
    ba.avg_ns / 1e6              AS ns_avg_7d_mln,
    ba.avg_sr / 1e6              AS dep_avg_7d_mln,
    (ba.avg_dv + ba.avg_ns + ba.avg_sr) / 1e6
                                  AS total_avg_7d_mln
FROM bucket_avg ba
ORDER BY
    CASE ba.bucket
        WHEN N'0 / null'  THEN 0
        WHEN N'< 5 тыс'   THEN 1
        WHEN N'< 10 тыс'  THEN 2
        WHEN N'< 50 тыс'  THEN 3
        WHEN N'< 150 тыс' THEN 4
        WHEN N'< 500 тыс' THEN 5
        ELSE                    6   -- ≥ 500 тыс
    END;
