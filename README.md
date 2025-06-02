/*─────────────────────────────────────────────────────────────────────────────
  «Скользящая неделя»: для каждой из 7 дат
      1) относим КАЖДОГО «розничного» клиента к бакету
         ─ бакет определяется ДВС-остатком на ЭТУ дату
      2) складываем портфель бакета (ДВС / НС / Срочные)
  В финале считаем СРЕДНЕЕ (AVG) по 7 датам для каждого бакета.

  ─ валюта      : RUB (CUR = 810)
  ─ секции      : До востребования / Накопительный счёт / Срочные
  ─ окно        : @rep_date − @days_back … @rep_date   (по-умолчанию 7 сут.)
  ─ бакеты      : 0|NULL, <5k, <10k, <50k, <150k, <500k, ≥500k
─────────────────────────────────────────────────────────────────────────────*/
DECLARE @rep_date  date = '2025-05-11';     -- конец окна
DECLARE @days_back int  = 6;                -- 7 дат в выборке
/*---------------------------------------------------------------------------*/

;WITH daily_cli AS (         /* 1. Остатки клиента по секциям за каждую дату */
    SELECT
        db.dt_rep,
        db.cli_id,
        SUM(CASE WHEN TRIM(db.section_name) = N'До востребования'
                 THEN db.out_rub END)                       AS dv_rub,
        SUM(CASE WHEN TRIM(db.section_name) = N'Накопительный счёт'
                 THEN db.out_rub END)                       AS ns_rub,
        SUM(CASE WHEN TRIM(db.section_name) = N'Срочные'
                 THEN db.out_rub END)                       AS sr_rub
    FROM ALM.balance_rest_all db WITH (NOLOCK)
    WHERE db.dt_rep BETWEEN DATEADD(day,-@days_back,@rep_date) AND @rep_date
      AND db.CUR          = '810'
      AND db.MAP_IS_CASH  = 1
      AND db.TSEGMENTNAME = N'Розничный бизнес'
      AND db.AP           = N'Пассив'
      AND db.BLOCK_NAME   = N'Привлечение ФЛ'
      AND TRIM(db.section_name) IN (N'Срочные',
                                    N'До востребования',
                                    N'Накопительный счёт')
    GROUP BY db.dt_rep, db.cli_id
),

bucket_day AS (             /* 2. Назначаем бакет по ДВС-остатку ЭТОГО дня   */
    SELECT
        dc.dt_rep,
        dc.cli_id,
        dc.dv_rub,
        dc.ns_rub,
        dc.sr_rub,
        CASE
            WHEN dc.dv_rub IS NULL OR dc.dv_rub <=     0      THEN N'0 / null'
            WHEN dc.dv_rub <       5000                      THEN N'< 5 тыс'
            WHEN dc.dv_rub <      10000                      THEN N'< 10 тыс'
            WHEN dc.dv_rub <      50000                      THEN N'< 50 тыс'
            WHEN dc.dv_rub <     150000                      THEN N'< 150 тыс'
            WHEN dc.dv_rub <     500000                      THEN N'< 500 тыс'
            ELSE                                               N'≥ 500 тыс'
        END AS bucket
    FROM daily_cli dc
),

bucket_tot AS (             /* 3. Итог по бакету в каждую дату              */
    SELECT
        bd.dt_rep,
        bd.bucket,
        COUNT(DISTINCT bd.cli_id)                                   AS clients_day,
        SUM(bd.dv_rub)                                              AS dv_sum_day,
        SUM(bd.ns_rub)                                              AS ns_sum_day,
        SUM(bd.sr_rub)                                              AS sr_sum_day
    FROM bucket_day bd
    GROUP BY bd.dt_rep, bd.bucket
),

bucket_avg AS (             /* 4. Среднее по 7 датам для каждого бакета     */
    SELECT
        bt.bucket,
        AVG(bt.clients_day)                                   AS avg_clients,
        AVG(bt.dv_sum_day)                                    AS avg_dv,
        AVG(bt.ns_sum_day)                                    AS avg_ns,
        AVG(bt.sr_sum_day)                                    AS avg_sr
    FROM bucket_tot bt
    GROUP BY bt.bucket
)

SELECT                      /* 5. Финал: выводим в млн ₽ + сортировка       */
    ba.bucket                               AS [Бакет (по ДВС на дату)],

    CAST(ba.avg_clients AS decimal(18,2))   AS avg_clients_day,

    ba.avg_dv / 1e6        AS dv_avg_7d_mln,
    ba.avg_ns / 1e6        AS ns_avg_7d_mln,
    ba.avg_sr / 1e6        AS dep_avg_7d_mln,
    (ba.avg_dv + ba.avg_ns + ba.avg_sr) / 1e6 AS total_avg_7d_mln

FROM bucket_avg ba
ORDER BY
    CASE ba.bucket
        WHEN N'0 / null'  THEN 0
        WHEN N'< 5 тыс'   THEN 1
        WHEN N'< 10 тыс'  THEN 2
        WHEN N'< 50 тыс'  THEN 3
        WHEN N'< 150 тыс' THEN 4
        WHEN N'< 500 тыс' THEN 5
        ELSE                    6
    END;
