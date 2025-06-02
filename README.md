/* -------------------------------------------------------------------------
   Параметры: отчётная дата и глубина «недели»
---------------------------------------------------------------------------*/
DECLARE @rep_date  date = '2025-05-11';
DECLARE @days_back int  = 6;   -- 6 дней + сама дата = 7 суток

/* 1. Балансы клиентов на отчётную дату ------------------------------------*/
;WITH current_bal AS (
    SELECT
        cli_id,
        SUM(CASE WHEN section_name = N'До востребования'   THEN out_rub END) AS dv_rub,
        SUM(CASE WHEN section_name = N'Накопительный счёт' THEN out_rub END) AS ns_rub,
        SUM(CASE WHEN section_name = N'Срочные'            THEN out_rub END) AS sr_rub,
        SUM(out_rub)                                          AS total_rub
    FROM ALM.balance_rest_all WITH (NOLOCK)
    WHERE dt_rep       = @rep_date
      AND MAP_IS_CASH  = 1
      AND TSEGMENTNAME = N'Розничный бизнес'
      AND AP           = N'Пассив'
      AND BLOCK_NAME   = N'Привлечение ФЛ'
      AND section_name IN (N'Срочные', N'До востребования', N'Накопительный счёт')
    GROUP BY cli_id
),

/* 2. Присваиваем клиенту бакет по сумме ДВС -------------------------------*/
bucketed AS (
    SELECT
        cli_id,
        dv_rub, ns_rub, sr_rub, total_rub,
        CASE
            WHEN dv_rub IS NULL OR dv_rub <=      0         THEN N'0 / null'
            WHEN dv_rub <=     10000                                THEN N'≤ 10 тыс'
            WHEN dv_rub <=     50000                                THEN N'≤ 50 тыс'
            WHEN dv_rub <=    150000                                THEN N'≤ 150 тыс'
            WHEN dv_rub <=    500000                                THEN N'≤ 500 тыс'
            ELSE                                                     N'> 500 тыс'
        END AS bucket
    FROM current_bal
),

/* 3. Средние остатки клиента за последние 7 суток -------------------------*/
weekly_avg AS (
    SELECT
        cli_id,
        AVG(CASE WHEN section_name = N'До востребования'   THEN out_rub END) AS avg_dv,
        AVG(CASE WHEN section_name = N'Накопительный счёт' THEN out_rub END) AS avg_ns,
        AVG(CASE WHEN section_name = N'Срочные'            THEN out_rub END) AS avg_sr
    FROM ALM.balance_rest_all WITH (NOLOCK)
    WHERE dt_rep BETWEEN DATEADD(day,-@days_back, @rep_date) AND @rep_date
      AND MAP_IS_CASH  = 1
      AND TSEGMENTNAME = N'Розничный бизнес'
      AND AP           = N'Пассив'
      AND BLOCK_NAME   = N'Привлечение ФЛ'
      AND section_name IN (N'Срочные', N'До востребования', N'Накопительный счёт')
    GROUP BY cli_id
),

/* 4. Склеиваем факт дня и недельные средние -------------------------------*/
joined AS (
    SELECT
        b.bucket,
        b.cli_id,

        b.dv_rub, b.ns_rub, b.sr_rub, b.total_rub,

        COALESCE(w.avg_dv,0)                       AS avg_dv,
        COALESCE(w.avg_ns,0)                       AS avg_ns,
        COALESCE(w.avg_sr,0)                       AS avg_sr,
        COALESCE(w.avg_dv,0)+COALESCE(w.avg_ns,0)+
        COALESCE(w.avg_sr,0)                       AS avg_total
    FROM bucketed   b
    LEFT JOIN weekly_avg w ON w.cli_id = b.cli_id
)

/* 5. Итог по бакетам + «суммы средних» ------------------------------------*/
SELECT
    bucket                                               AS [Бакет ДВС],
    COUNT(DISTINCT cli_id)                               AS [Клиентов],

    /* --- срез на дату --------------------------------------------------- */
    SUM(total_rub) / 1e6  AS [Сумма-дата, млн],
    SUM(dv_rub)   / 1e6   AS [ДВС-дата, млн],
    SUM(ns_rub)   / 1e6   AS [НС-дата,  млн],
    SUM(sr_rub)   / 1e6   AS [Вклады-дата, млн],

    /* --- сумма недельных-средних --------------------------------------- */
    SUM(avg_total) / 1e6  AS [Сумма-7д, млн],
    SUM(avg_dv)    / 1e6  AS [ДВС-7д,   млн],
    SUM(avg_ns)    / 1e6  AS [НС-7д,    млн],
    SUM(avg_sr)    / 1e6  AS [Вклады-7д, млн]

FROM joined
GROUP BY bucket
ORDER BY
    CASE bucket
        WHEN N'0 / null'  THEN 0
        WHEN N'≤ 10 тыс'  THEN 1
        WHEN N'≤ 50 тыс'  THEN 2
        WHEN N'≤ 150 тыс' THEN 3
        WHEN N'≤ 500 тыс' THEN 4
        ELSE                      5
    END;
