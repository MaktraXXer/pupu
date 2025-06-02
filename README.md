/*─────────────────────────────────────────────────────────────────────────────
   ЕДИНЫЙ СКРИПТ: «розничные» клиенты, распределённые по бакетам
   (бакет строится по СРЕДНЕМУ ДВС-остатку клиента за последние 7 суток)

   – валюта                 : только RUB  (CUR = 810)
   – секции                 : ДВС / НС / Срочные  (названия нормализуем TRIM’ом)
   – окно для средних       : @rep_date – @days_back … @rep_date  включительно
   – бакеты по avg(ДВС)     : 0/null, <5 k, <10 k, <50 k, <150 k, <500 k, ≥500 k
   – выводим:
        • clients_week      – сколько клиентов участвовало в расчёте средних
        • clients_live      – сколько из них имеют рубли на дату @rep_date
        • суммы средних за 7 суток  (портфель и по трём секциям)
        • средний остаток на 1 клиента (млн ₽) за 7 суток
        • при желании: суммы «на дату»  (закомментированные колонки можно включить)

   Платформа: Microsoft SQL Server 2017+
─────────────────────────────────────────────────────────────────────────────*/
DECLARE @rep_date  date = '2025-05-11';  -- отчётная дата
DECLARE @days_back int  = 6;             -- 6 дней назад + сама дата = 7 суток

/* 1. Суточные остатки клиента по секциям ───────────────────────────────────*/
;WITH daily_bal AS (
    SELECT
        db.dt_rep,
        db.cli_id,
        SUM(CASE WHEN TRIM(db.section_name) = N'До востребования'   THEN db.out_rub END) AS dv_rub,
        SUM(CASE WHEN TRIM(db.section_name) = N'Накопительный счёт' THEN db.out_rub END) AS ns_rub,
        SUM(CASE WHEN TRIM(db.section_name) = N'Срочные'            THEN db.out_rub END) AS sr_rub
    FROM ALM.balance_rest_all db WITH (NOLOCK)
    WHERE db.dt_rep BETWEEN DATEADD(day,-@days_back,@rep_date) AND @rep_date
      AND db.CUR          = '810'                      -- только рубли (RUB 810)
      AND db.MAP_IS_CASH  = 1
      AND db.TSEGMENTNAME = N'Розничный бизнес'
      AND db.AP           = N'Пассив'
      AND db.BLOCK_NAME   = N'Привлечение ФЛ'
      AND TRIM(db.section_name) IN (N'Срочные',
                                    N'До востребования',
                                    N'Накопительный счёт')
    GROUP BY db.dt_rep, db.cli_id
),

/* 2. Среднее «пока деньги > 0» за 7 суток ──────────────────────────────────*/
weekly_avg AS (
    SELECT
        cli_id,
        AVG(NULLIF(dv_rub,0)) AS avg_dv,   -- нули в расчёт не берём
        AVG(NULLIF(ns_rub,0)) AS avg_ns,
        AVG(NULLIF(sr_rub,0)) AS avg_sr
    FROM daily_bal
    GROUP BY cli_id
),

/* 3. Бакет по среднему ДВС ─────────────────────────────────────────────────*/
bucketed_avg AS (
    SELECT
        wa.cli_id,
        wa.avg_dv,
        wa.avg_ns,
        wa.avg_sr,
        COALESCE(wa.avg_dv,0) + COALESCE(wa.avg_ns,0) + COALESCE(wa.avg_sr,0) AS avg_total,

        CASE
            WHEN wa.avg_dv IS NULL OR wa.avg_dv <=      0     THEN N'0 / null'
            WHEN wa.avg_dv <        5000                      THEN N'< 5 тыс'
            WHEN wa.avg_dv <       10000                      THEN N'< 10 тыс'
            WHEN wa.avg_dv <       50000                      THEN N'< 50 тыс'
            WHEN wa.avg_dv <      150000                      THEN N'< 150 тыс'
            WHEN wa.avg_dv <      500000                      THEN N'< 500 тыс'
            ELSE                                                N'≥ 500 тыс'
        END AS bucket
    FROM weekly_avg wa
),

/* 4. Живые на дату @rep_date (для clients_live) ────────────────────────────*/
current_clients AS (
    SELECT DISTINCT cb.cli_id
    FROM ALM.balance_rest_all cb WITH (NOLOCK)
    WHERE cb.dt_rep       = @rep_date
      AND cb.CUR          = '810'
      AND cb.MAP_IS_CASH  = 1
      AND cb.TSEGMENTNAME = N'Розничный бизнес'
      AND cb.AP           = N'Пассив'
      AND cb.BLOCK_NAME   = N'Привлечение ФЛ'
      AND TRIM(cb.section_name) IN (N'Срочные',
                                    N'До востребования',
                                    N'Накопительный счёт')
)

/* 5. Итоговая сводка по бакетам ────────────────────────────────────────────*/
SELECT
    b.bucket                                               AS [Бакет avg(ДВС-7д)],

    /* клиентов */
    COUNT(DISTINCT b.cli_id)                               AS clients_week,       -- участвовали в 7-сут. расчёте
    COUNT(DISTINCT cc.cli_id)                              AS clients_live,       -- имеют рубли на @rep_date

    /* суммы средних за неделю (млн ₽) */
    SUM( COALESCE(b.avg_total,0) ) / 1e6                   AS sum_avg_7d_mln,
    SUM( COALESCE(b.avg_dv,0)   ) / 1e6                    AS dv_avg_7d_mln,
    SUM( COALESCE(b.avg_ns,0)   ) / 1e6                    AS ns_avg_7d_mln,
    SUM( COALESCE(b.avg_sr,0)   ) / 1e6                    AS dep_avg_7d_mln,

    /* средний остаток на 1 клиента (млн ₽) */
    ROUND( SUM( COALESCE(b.avg_dv,0) )  /
           NULLIF( COUNT(DISTINCT b.cli_id), 0 ) / 1e6 , 4 )   AS avg_dv_per_cli_mln,
    ROUND( SUM( COALESCE(b.avg_ns,0) )  /
           NULLIF( COUNT(DISTINCT b.cli_id), 0 ) / 1e6 , 4 )   AS avg_ns_per_cli_mln,
    ROUND( SUM( COALESCE(b.avg_sr,0) )  /
           NULLIF( COUNT(DISTINCT b.cli_id), 0 ) / 1e6 , 4 )   AS avg_dep_per_cli_mln,
    ROUND( SUM( COALESCE(b.avg_total,0) ) /
           NULLIF( COUNT(DISTINCT b.cli_id), 0 ) / 1e6 , 4 )   AS avg_total_per_cli_mln

    /* ─ при необходимости можно раскомментировать блок ниже,
         чтобы дополнительно вывести портфель «на дату» ─
  , SUM( c.total_rub ) / 1e6  AS sum_date_mln
  , SUM( c.dv_rub )    / 1e6  AS dv_date_mln
  , SUM( c.ns_rub )    / 1e6  AS ns_date_mln
  , SUM( c.sr_rub )    / 1e6  AS dep_date_mln
    */

FROM bucketed_avg b
LEFT JOIN current_clients cc ON cc.cli_id = b.cli_id
/*LEFT JOIN current_bal c     ON c.cli_id = b.cli_id*/   -- если нужен блок «на дату»
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
