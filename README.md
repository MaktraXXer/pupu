/* -------------------------------------------------------------------------
   Параметры: дата среза и глубина для «недели» (можно менять при вызове)
---------------------------------------------------------------------------*/
DECLARE @rep_date  date = '2025-05-11';   -- актуальная отчётная дата
DECLARE @days_back int  = 6;              -- 6 дней назад + сама дата = 7 суток

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

/* 3. Средний ДВС-остаток клиента за 7 суток -------------------------------*/
weekly_avg AS (
    SELECT
        cli_id,
        AVG(out_rub) AS avg_dv_week
    FROM ALM.balance_rest_all WITH (NOLOCK)
    WHERE dt_rep BETWEEN DATEADD(day,-@days_back,@rep_date) AND @rep_date
      AND MAP_IS_CASH  = 1
      AND TSEGMENTNAME = N'Розничный бизнес'
      AND AP           = N'Пассив'
      AND BLOCK_NAME   = N'Привлечение ФЛ'
      AND section_name = N'До востребования'
    GROUP BY cli_id
),

/* 4. Объединяем факт дня и среднюю неделю ---------------------------------*/
joined AS (
    SELECT
        b.bucket,
        b.cli_id,
        b.dv_rub, b.ns_rub, b.sr_rub, b.total_rub,
        w.avg_dv_week
    FROM bucketed   b
    LEFT JOIN weekly_avg w ON w.cli_id = b.cli_id
)

/* 5. Итоговая сводка по бакетам -------------------------------------------*/
SELECT
    bucket                                               AS [Бакет ДВС],
    COUNT(DISTINCT cli_id)                               AS [Клиентов],
    SUM(total_rub)   / 1e6                               AS [Сумма, млн],
    SUM(dv_rub)      / 1e6                               AS [ДВС, млн],
    SUM(ns_rub)      / 1e6                               AS [НС, млн],
    SUM(sr_rub)      / 1e6                               AS [Срочные, млн],
    AVG(avg_dv_week) / 1e6                               AS [Средний ДВС-7д, млн]
FROM joined
GROUP BY bucket
ORDER BY
    CASE bucket                  -- фиксируем порядок вывода
        WHEN N'0 / null'  THEN 0
        WHEN N'≤ 10 тыс'  THEN 1
        WHEN N'≤ 50 тыс'  THEN 2
        WHEN N'≤ 150 тыс' THEN 3
        WHEN N'≤ 500 тыс' THEN 4
        ELSE                      5
    END;
