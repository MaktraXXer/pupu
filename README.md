/* -------------------------------------------------------------
   Сравнение актуальной таблицы депозитов с архивом
   за произвольный двухнедельный промежуток + две «особые» даты
   ------------------------------------------------------------- */
WITH base_data AS (          /* исходные записи по контрактам */
    SELECT DISTINCT
        dep.dt_rep,
        dep.con_id,
        dep.balance_rub
    FROM LIQUIDITY.liq.DepositInterestsRate dep WITH (NOLOCK)
    WHERE dep.cli_subtype = 'ФЛ'
      AND dep.isfloat     <> 1
      AND dep.cur          = 'RUR'
),

/* список календарных дат, которые нужно проверить
   (22 дня начиная с 16-июн-2025) */
report_dates AS (
    SELECT TOP (22)
           DATEADD(DAY, ROW_NUMBER() OVER (ORDER BY number) - 1,
                   '2025-06-16') AS report_date
    FROM master.dbo.spt_values
    WHERE type = 'P'                         -- гарантируем набор чисел
    ORDER BY number
),

/* остатки/выборки по контрактам на момент отчётных дат */
saldo_data AS (
    SELECT
        bd.con_id,
        s.out_rub,
        bd.dt_rep
    FROM base_data bd
    LEFT JOIN LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
           ON  bd.con_id = s.con_id
          AND bd.dt_rep BETWEEN s.dt_from AND s.dt_to
),

/* агрегаты по каждому дню */
stats_for_each_day AS (
    SELECT
        rd.report_date,
        COUNT(bd.con_id)                                   AS cnt_con_id,
        SUM(ISNULL(bd.balance_rub, 0))                     AS sum_balance_rub,
        COUNT(CASE WHEN sd.out_rub IS NOT NULL THEN 1 END) AS cnt_with_out_rub,
        COUNT(CASE WHEN sd.out_rub IS NULL  THEN 1 END)    AS cnt_without_out_rub
    FROM report_dates rd
    LEFT JOIN base_data bd
           ON bd.dt_rep = rd.report_date
    LEFT JOIN saldo_data sd
           ON bd.con_id = sd.con_id
          AND bd.dt_rep = sd.dt_rep
    GROUP BY rd.report_date
),

/* две «особые» даты:
   – 08-июл-2025
   – последняя доступная dt_rep                                     */
special_reports AS (
    /* 08-июл-2025 */
    SELECT
        CAST('2025-07-08' AS date) AS special_dt_rep,
        COUNT(*)                   AS cnt_con_id_20250708,
        SUM(balance_rub)           AS sum_balance_rub_20250708,
        NULL                       AS cnt_con_id_max_dt_rep,
        NULL                       AS sum_balance_rub_max_dt_rep
    FROM base_data
    WHERE dt_rep = '2025-07-08'

    UNION ALL

    /* самая поздняя дата в источнике */
    SELECT
        (SELECT MAX(dt_rep)
         FROM LIQUIDITY.liq.DepositInterestsRate WITH (NOLOCK)) AS special_dt_rep,
        NULL                       AS cnt_con_id_20250708,
        NULL                       AS sum_balance_rub_20250708,
        COUNT(*)                   AS cnt_con_id_max_dt_rep,
        SUM(balance_rub)           AS sum_balance_rub_max_dt_rep
    FROM base_data
    WHERE dt_rep = (SELECT MAX(dt_rep)
                    FROM LIQUIDITY.liq.DepositInterestsRate WITH (NOLOCK))
)

/* итоговый вывод */
SELECT
    COALESCE(sr.special_dt_rep, sfd.report_date) AS report_date,
    sr.cnt_con_id_20250708,
    sr.sum_balance_rub_20250708,
    sr.cnt_con_id_max_dt_rep,
    sr.sum_balance_rub_max_dt_rep,
    sfd.cnt_con_id,
    sfd.sum_balance_rub,
    sfd.cnt_with_out_rub,
    sfd.cnt_without_out_rub
FROM stats_for_each_day sfd
FULL OUTER JOIN special_reports sr
       ON sr.special_dt_rep = sfd.report_date
ORDER BY COALESCE(sr.special_dt_rep, sfd.report_date);
