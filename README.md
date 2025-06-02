DECLARE @rep_date  date = '2025-05-11';  -- отчётная дата
DECLARE @days_back int  = 6;             -- 7-сут. окно: @rep_date – 6 … @rep_date

/* 1. Суточные остатки клиента по секциям ----------------------------------*/
;WITH daily_bal AS (
    SELECT
        dt_rep,
        cli_id,
        SUM(CASE WHEN section_name = N'До востребования'   THEN out_rub END) AS dv_rub,
        SUM(CASE WHEN section_name = N'Накопительный счёт' THEN out_rub END) AS ns_rub,
        SUM(CASE WHEN section_name = N'Срочные'            THEN out_rub END) AS sr_rub
    FROM ALM.balance_rest_all WITH (NOLOCK)
    WHERE dt_rep BETWEEN DATEADD(day, -@days_back, @rep_date) AND @rep_date
      AND CUR          = '810'                         -- <<< новый фильтр
      AND MAP_IS_CASH  = 1
      AND TSEGMENTNAME = N'Розничный бизнес'
      AND AP           = N'Пассив'
      AND BLOCK_NAME   = N'Привлечение ФЛ'
      AND section_name IN (N'Срочные', N'До востребования', N'Накопительный счёт')
    GROUP BY dt_rep, cli_id
),

/* 2. Средние дневные остатки клиента за 7 суток ----------------------------*/
weekly_avg AS (
    SELECT
        cli_id,
        AVG(dv_rub) AS avg_dv,
        AVG(ns_rub) AS avg_ns,
        AVG(sr_rub) AS avg_sr
    FROM daily_bal
    GROUP BY cli_id
),

/* 3. Бакет по среднему ДВС-остатку ----------------------------------------*/
bucketed_avg AS (
    SELECT
        cli_id,
        avg_dv,
        avg_ns,
        avg_sr,
        COALESCE(avg_dv,0) + COALESCE(avg_ns,0) + COALESCE(avg_sr,0) AS avg_total,
        CASE
            WHEN avg_dv IS NULL OR avg_dv <=     0      THEN N'0 / null'
            WHEN avg_dv <       5000                 THEN N'< 5 тыс'
            WHEN avg_dv <      10000                 THEN N'< 10 тыс'
            WHEN avg_dv <      50000                 THEN N'< 50 тыс'
            WHEN avg_dv <     150000                 THEN N'< 150 тыс'
            WHEN avg_dv <     500000                 THEN N'< 500 тыс'
            ELSE                                         N'≥ 500 тыс'
        END AS bucket
    FROM weekly_avg
),

/* 4. Баланс клиента на отчётную дату (для справки) ------------------------*/
current_bal AS (
    SELECT
        cli_id,
        SUM(CASE WHEN section_name = N'До востребования'   THEN out_rub END) AS dv_rub,
        SUM(CASE WHEN section_name = N'Накопительный счёт' THEN out_rub END) AS ns_rub,
        SUM(CASE WHEN section_name = N'Срочные'            THEN out_rub END) AS sr_rub,
        SUM(out_rub) AS total_rub
    FROM ALM.balance_rest_all WITH (NOLOCK)
    WHERE dt_rep       = @rep_date
      AND CUR          = '810'                         -- <<< тот же фильтр
      AND MAP_IS_CASH  = 1
      AND TSEGMENTNAME = N'Розничный бизнес'
      AND AP           = N'Пассив'
      AND BLOCK_NAME   = N'Привлечение ФЛ'
      AND section_name IN (N'Срочные', N'До востребования', N'Накопительный счёт')
    GROUP BY cli_id
)

/* 5. Итоговая сводка по бакету -------------------------------------------*/
SELECT
    b.bucket                                                 AS [Бакет (avg ДВС-7д)],
    COUNT(DISTINCT b.cli_id)                                 AS [Клиентов],

    /* ---------- сумма средних за 7 суток ---------- */
    SUM(COALESCE(b.avg_dv,0) + COALESCE(b.avg_ns,0) + COALESCE(b.avg_sr,0)) / 1e6
        AS [Сумма-7д, млн],
    SUM(b.avg_dv) / 1e6  AS [ДВС-7д,   млн],
    SUM(b.avg_ns) / 1e6  AS [НС-7д,    млн],
    SUM(b.avg_sr) / 1e6  AS [Вклады-7д, млн],

    /* ---------- срез на дату (опционально) ---------- */
    SUM(c.total_rub) / 1e6  AS [Сумма-дата, млн],
    SUM(c.dv_rub)   / 1e6   AS [ДВС-дата,   млн],
    SUM(c.ns_rub)   / 1e6   AS [НС-дата,    млн],
    SUM(c.sr_rub)   / 1e6   AS [Вклады-дата, млн]

FROM bucketed_avg b
LEFT JOIN current_bal c ON c.cli_id = b.cli_id
GROUP BY b.bucket
ORDER BY
    CASE b.bucket
        WHEN N'0 / null'  THEN 0
        WHEN N'< 5 тыс'   THEN 1
        WHEN N'< 10 тыс'  THEN 2
        WHEN N'< 50 тыс'  THEN 3
        WHEN N'< 150 тыс' THEN 4
        WHEN N'< 500 тыс' THEN 5
        ELSE                    6          -- ≥ 500 тыс
    END;
