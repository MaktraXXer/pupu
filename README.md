Ниже пример-запроса, который:

* берёт «подозрительные» `con_id`, впервые вошедшие **как новые** в ДВС-портфель 18-30 марта 2025;
* строит «пря­моуголь­ные» интервалы, в которых сегмент (`section_name`) оставался без изменений;
* показывает, **каким был сегмент до первого переключения, на какой сегмент он изменился, с какой по какую дату этот «неправильный» сегмент держался, когда и на что сменился обратно**,
* а также даёт сегмент и сумму остатка (`out_rub`) этих договоров на `2025-04-01`.

```sql
/* ----------  Параметры  ---------- */
DECLARE @dvs_from  date =  N'2025-03-18';
DECLARE @dvs_to    date =  N'2025-03-30';
DECLARE @period_from date = N'2025-03-01';
DECLARE @period_to   date = N'2025-04-01';  -- включительно

/* ---------- 1. con_id, впервые «новые» в ДВС 18-30 марта ---------- */
WITH dvs_new AS (
    SELECT DISTINCT con_id
    FROM alm.ALM.vw_balance_rest_all
    WHERE dt_rep BETWEEN @dvs_from AND @dvs_to
      AND section_name = N'До востребования'
      AND dt_open      = dt_rep            -- «новый»
      AND block_name   = N'Привлечение ФЛ'
      AND od_flag      = 1
      AND cur          = '810'
),

/* ---------- 2. все записи за март-апрель по этим con_id ---------- */
base AS (
    SELECT t.con_id,
           t.dt_rep,
           t.section_name,
           t.out_rub
    FROM alm.ALM.vw_balance_rest_all AS t
    JOIN dvs_new d  ON d.con_id = t.con_id
    WHERE t.dt_rep BETWEEN @period_from AND @period_to
      AND t.block_name = N'Привлечение ФЛ'
      AND t.od_flag    = 1
      AND t.cur        = '810'
),

/* ---------- 3. нумеруем «куски» одинаковых сегментов ---------- */
runs AS (
    SELECT b.*,
           /* 1, когда сегмент сменился относительно предыдущего дня */
           CASE WHEN LAG(section_name) OVER (PARTITION BY con_id ORDER BY dt_rep)
                      = section_name
                THEN 0 ELSE 1 END                                          AS seg_flip,
           /* cumulative-sum → ID «куска» одинаковых сегментов */
           SUM(CASE WHEN LAG(section_name) OVER (PARTITION BY con_id ORDER BY dt_rep)
                         = section_name
                    THEN 0 ELSE 1 END) 
           OVER (PARTITION BY con_id ORDER BY dt_rep ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
           AS run_id
    FROM base b
),

/* ---------- 4. агрегируем интервалы ---------- */
intervals AS (
    SELECT
        con_id,
        run_id,
        MIN(dt_rep)                      AS run_from,
        MAX(dt_rep)                      AS run_to,
        MIN(section_name)                AS section_name   -- одинаков внутри «куска»
    FROM runs
    GROUP BY con_id, run_id
),

/* ---------- 5. стыкуем интервалы и ищем «неправильный» кусок ---------- */
chg AS (
    SELECT
        cur.con_id,
        prev.section_name  AS segment_before,   -- что было до
        cur.section_name   AS segment_changed,  -- на что поменялось
        cur.run_from       AS changed_from,
        cur.run_to         AS changed_to,
        next.run_from      AS restored_from,    -- дата обратного переключения
        next.section_name  AS segment_after     -- что стало после
    FROM intervals cur
    LEFT JOIN intervals prev
           ON prev.con_id = cur.con_id
          AND prev.run_id = cur.run_id - 1
    LEFT JOIN intervals next
           ON next.con_id = cur.con_id
          AND next.run_id = cur.run_id + 1
    WHERE cur.segment_name = N'Накопительный счёт'     -- «чужой» сегмент
)

/* ---------- 6. сегмент и остаток этих договоров на 1 апреля ---------- */
SELECT
    c.con_id,
    c.segment_before,
    c.segment_changed,
    c.changed_from,
    c.changed_to,
    c.restored_from,
    c.segment_after,
    apr_1.section_name           AS segment_on_1apr,
    apr_1.out_rub                AS out_rub_on_1apr
FROM chg c
LEFT JOIN base apr_1
       ON apr_1.con_id = c.con_id
      AND apr_1.dt_rep = N'2025-04-01'
ORDER BY c.con_id;
```

**Что показывает результат**

| колонка                      | содержание                                         |
| ---------------------------- | -------------------------------------------------- |
| `con_id`                     | договор-идентификатор                              |
| `segment_before`             | сегмент, в котором договор находился **до** «сбоя» |
| `segment_changed`            | «чужой» сегмент, куда он внезапно попал            |
| `changed_from`, `changed_to` | период, пока договор числился в «чужом» сегменте   |
| `restored_from`              | дата, когда сегмент снова изменился                |
| `segment_after`              | сегмент, установленный после восстановления        |
| `segment_on_1apr`            | фактический сегмент на 1 апреля 2025               |
| `out_rub_on_1apr`            | остаток по договору на 1 апреля                    |

> **Если нужно ограничить вывод парой-тройкой примеров**, добавьте
> `TOP (N)` к последнему `SELECT` или дополните `WHERE c.con_id IN (…)`.

Так вы быстро увидите, у каких договоров ДВС-портфель «подхватил» накопительные счета, сколько дней это длилось и к какому состоянию они пришли к 1 апреля 2025.
