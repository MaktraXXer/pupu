/*─────────────────────────────────────────────────────────────────────────────
  «Розничные» клиенты, которые БЫЛИ АКТИВНЫ (имели ≥ 1 руб. в любой секции)
  КАЖДЫЙ из 7 календарных дней  – распределяем их по бакетам,
  считаем средние дневные остатки и среднее-на-клиента.

  условия:
    • валюта       = 810 (RUB)
    • секции       = До востребования / Накопительный счёт / Срочные
    • окно         = @rep_date − @days_back … @rep_date  (по-умолчанию 7 суток)
    • бакет        = по среднему ДВС-остатку клиента за 7 суток
                     0|NULL, <5k, <10k, <50k, <150k, <500k, ≥500k
─────────────────────────────────────────────────────────────────────────────*/
DECLARE @rep_date  date = '2025-05-11';   -- конец окна
DECLARE @days_back int  = 6;              -- 6 дней назад  → всего 7 суток
/*---------------------------------------------------------------------------*/

;WITH daily_bal AS (          /* 1. Суточные портфели клиента */
    SELECT
        db.dt_rep,
        db.cli_id,
        SUM(CASE WHEN LTRIM(RTRIM(db.section_name)) = N'До востребования'
                 THEN db.out_rub END)                             AS dv_rub,
        SUM(CASE WHEN LTRIM(RTRIM(db.section_name)) = N'Накопительный счёт'
                 THEN db.out_rub END)                             AS ns_rub,
        SUM(CASE WHEN LTRIM(RTRIM(db.section_name)) = N'Срочные'
                 THEN db.out_rub END)                             AS sr_rub
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

valid_clients AS (           /* 2. Оставляем ТОЛЬКО тех, у кого 7 дат с >0 руб */
    SELECT cli_id
    FROM daily_bal
    WHERE ISNULL(dv_rub,0) + ISNULL(ns_rub,0) + ISNULL(sr_rub,0) > 0
    GROUP BY cli_id
    HAVING COUNT(DISTINCT dt_rep) = @days_back + 1        -- все 7 дней
),

weekly_avg AS (              /* 3. Средние дневные остатки за 7 суток */
    SELECT
        d.cli_id,
        AVG(d.dv_rub)                          AS avg_dv,
        AVG(d.ns_rub)                          AS avg_ns,
        AVG(d.sr_rub)                          AS avg_sr
    FROM daily_bal d
    JOIN valid_clients v  ON v.cli_id = d.cli_id
    GROUP BY d.cli_id
),

bucketed_avg AS (            /* 4. Назначаем бакет по avg_dv */
    SELECT
        wa.cli_id,
        wa.avg_dv,
        wa.avg_ns,
        wa.avg_sr,
        wa.avg_dv + wa.avg_ns + wa.avg_sr      AS avg_total,

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
)

SELECT                          /* 5. Итог по бакетам */
    b.bucket                                        AS [Бакет avg(ДВС-7д)],

    COUNT(*)                                        AS clients_week,   -- все 100 % «живые 7/7»

    /* суммы средних за 7 суток (млн ₽) */
    SUM(b.avg_total)/1e6        AS sum_avg_7d_mln,
    SUM(b.avg_dv)/1e6           AS dv_avg_7d_mln,
    SUM(b.avg_ns)/1e6           AS ns_avg_7d_mln,
    SUM(b.avg_sr)/1e6           AS dep_avg_7d_mln,

    /* средний остаток на 1 клиента (млн ₽) */
    ROUND( SUM(b.avg_dv)   / NULLIF(COUNT(*),0) / 1e6 , 4 ) AS avg_dv_per_cli_mln,
    ROUND( SUM(b.avg_ns)   / NULLIF(COUNT(*),0) / 1e6 , 4 ) AS avg_ns_per_cli_mln,
    ROUND( SUM(b.avg_sr)   / NULLIF(COUNT(*),0) / 1e6 , 4 ) AS avg_dep_per_cli_mln,
    ROUND( SUM(b.avg_total)/ NULLIF(COUNT(*),0) / 1e6 , 4 ) AS avg_total_per_cli_mln

FROM bucketed_avg b
GROUP BY b.bucket
ORDER BY
    CASE b.bucket
        WHEN N'0 / null'  THEN 0
        WHEN N'< 5 тыс'   THEN 1
        WHEN N'< 10 тыс'  THEN 2
        WHEN N'< 50 тыс'  THEN 3
        WHEN N'< 150 тыс' THEN 4
        WHEN N'< 500 тыс' THEN 5
        ELSE                    6      -- ≥ 500 тыс
    END;
