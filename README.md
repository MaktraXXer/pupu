Ниже — один-проходный вариант без «гряда» дата × клиент.
Работает за счёт двух группировок вместо кросс-джойна, поэтому спокойно тянет окно 30 суток (или больше, если нужно).

/*========================================================================
  ПАРАМЕТРЫ РАСЧЁТА
========================================================================*/
DECLARE @rep_date  date = '2025-05-11';   -- последняя дата периода
DECLARE @days_back int  = 29;             -- 29 → вместе с @rep_date = 30 суток
DECLARE @total_days int = @days_back + 1; -- понадобится для среднего

/*========================================================================
  1. СРЕДНИЙ ДВС-ОСТАТОК КЛИЕНТА ЗА ПЕРИОД  →  БАКЕТ
     (одна агрегация по cli_id, без пустых дат - нули нам и не нужны:
      SUM(out_rub) / 30 даст тот же результат)
========================================================================*/
;WITH client_avg AS (
    SELECT
        cli_id,
        SUM(
            CASE WHEN LTRIM(RTRIM(section_name)) = N'До востребования'
                 THEN out_rub ELSE 0 END
        ) / @total_days                                      AS avg_dv_30d
    FROM ALM.balance_rest_all WITH (NOLOCK)
    WHERE dt_rep BETWEEN DATEADD(day,-@days_back,@rep_date) AND @rep_date
      AND CUR          = '810'
      AND MAP_IS_CASH  = 1
      AND TSEGMENTNAME = N'Розничный бизнес'
      AND AP           = N'Пассив'
      AND BLOCK_NAME   = N'Привлечение ФЛ'
      AND LTRIM(RTRIM(section_name)) IN
          (N'Срочные', N'До востребования', N'Накопительный счёт')
    GROUP BY cli_id
),
bucket_map AS (
    SELECT
        cli_id,
        CASE
             WHEN avg_dv_30d <=      0     THEN N'0 / null'
             WHEN avg_dv_30d <     5000    THEN N'< 5 тыс'
             WHEN avg_dv_30d <    10000    THEN N'< 10 тыс'
             WHEN avg_dv_30d <    50000    THEN N'< 50 тыс'
             WHEN avg_dv_30d <   150000    THEN N'< 150 тыс'
             WHEN avg_dv_30d <   500000    THEN N'< 500 тыс'
             ELSE                             N'≥ 500 тыс'
        END AS bucket
    FROM client_avg
),

/*========================================================================
  2. СУТОЧНЫЕ ОСТАТКИ (группируем только то, что реально есть в таблице)
========================================================================*/
daily_agg AS (
    SELECT
        dt_rep,
        cli_id,
        SUM(CASE WHEN LTRIM(RTRIM(section_name)) = N'До востребования'
                 THEN out_rub ELSE 0 END)   AS dv_rub,
        SUM(CASE WHEN LTRIM(RTRIM(section_name)) = N'Накопительный счёт'
                 THEN out_rub ELSE 0 END)   AS ns_rub,
        SUM(CASE WHEN LTRIM(RTRIM(section_name)) = N'Срочные'
                 THEN out_rub ELSE 0 END)   AS sr_rub
    FROM ALM.balance_rest_all WITH (NOLOCK)
    WHERE dt_rep BETWEEN DATEADD(day,-@days_back,@rep_date) AND @rep_date
      AND CUR          = '810'
      AND MAP_IS_CASH  = 1
      AND TSEGMENTNAME = N'Розничный бизнес'
      AND AP           = N'Пассив'
      AND BLOCK_NAME   = N'Привлечение ФЛ'
      AND LTRIM(RTRIM(section_name)) IN
          (N'Срочные', N'До востребования', N'Накопительный счёт')
    GROUP BY dt_rep, cli_id
),

/*========================================================================
  3. ПРИВЯЗЫВАЕМ БАКЕТ И СЧИТАЕМ ПО ДАТЕ + БАКЕТУ ШЕСТЬ ПОКАЗАТЕЛЕЙ
========================================================================*/
bucket_date AS (
    SELECT
        da.dt_rep,
        bm.bucket,

        /* --- счётчики клиентов ----------------------------------- */
        COUNT(DISTINCT CASE WHEN da.dv_rub > 0 THEN da.cli_id END) AS cnt_dv_cli,
        COUNT(DISTINCT CASE WHEN da.ns_rub > 0 THEN da.cli_id END) AS cnt_ns_cli,
        COUNT(DISTINCT CASE WHEN da.sr_rub > 0 THEN da.cli_id END) AS cnt_sr_cli,

        /* --- суммарные объёмы ------------------------------------ */
        SUM(CASE WHEN da.dv_rub > 0 THEN da.dv_rub END) AS sum_dv_rub,
        SUM(CASE WHEN da.ns_rub > 0 THEN da.ns_rub END) AS sum_ns_rub,
        SUM(CASE WHEN da.sr_rub > 0 THEN da.sr_rub END) AS sum_sr_rub
    FROM daily_agg   da
    JOIN bucket_map  bm  ON bm.cli_id = da.cli_id
    GROUP BY da.dt_rep, bm.bucket
)

/*========================================================================
  4. ФИНАЛ: пересчитываем средний объём на клиента прямо в SELECT
     (деление только в одной точке — так быстрее)
========================================================================*/
SELECT
    bd.dt_rep                                        AS [Дата],
    bd.bucket                                        AS [Бакет avg(ДВС-30д)],

    /* ---- До востребования -------------------------------------- */
    bd.cnt_dv_cli                                    AS cnt_dv_cli,
    CAST( CASE WHEN bd.cnt_dv_cli = 0
               THEN 0
               ELSE bd.sum_dv_rub / bd.cnt_dv_cli END
          AS decimal(18,2) )                         AS avg_dv_rub,

    /* ---- Накопительный счёт ------------------------------------ */
    bd.cnt_ns_cli                                    AS cnt_ns_cli,
    CAST( CASE WHEN bd.cnt_ns_cli = 0
               THEN 0
               ELSE bd.sum_ns_rub / bd.cnt_ns_cli END
          AS decimal(18,2) )                         AS avg_ns_rub,

    /* ---- Срочные ----------------------------------------------- */
    bd.cnt_sr_cli                                    AS cnt_sr_cli,
    CAST( CASE WHEN bd.cnt_sr_cli = 0
               THEN 0
               ELSE bd.sum_sr_rub / bd.cnt_sr_cli END
          AS decimal(18,2) )                         AS avg_sr_rub
FROM bucket_date bd
ORDER BY
    bd.dt_rep,
    CASE bd.bucket
        WHEN N'0 / null'  THEN 0
        WHEN N'< 5 тыс'   THEN 1
        WHEN N'< 10 тыс'  THEN 2
        WHEN N'< 50 тыс'  THEN 3
        WHEN N'< 150 тыс' THEN 4
        WHEN N'< 500 тыс' THEN 5
        ELSE                    6
    END
OPTION (RECOMPILE);   -- оптимизатор получит точные значения параметров

Почему этот вариант «легче»

Приём	Что даёт
Нет CROSS JOIN	Убрали «грязь» дата × клиент, поэтому объём строк = фактическим записям таблицы.
Два прохода по balance_rest_all	Первый — сразу считает недельный/месячный avg_dv (по cli_id), второй — суточные остатки (по dt_rep, cli_id). Оба пользуются одному диапазону дат, индекс dt_rep отрабатывает на 100 %.
Деление на @total_days внутри SUM	Для среднего не нужны «пустые» дни; нули и так не влияют на сумму.
OPTION (RECOMPILE)	План строится с конкретным диапазоном дат, лишние партиции не сканируются.

Меняете горизонт — задайте другое @days_back и @total_days (например, 59 → 60 дней).
Бакеты, названия секций, валюта — редактируются в одном месте.
