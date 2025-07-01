Отлично, задача понятна. Ниже — два запроса:

---

## **1. Найти `con_id`, которые есть в ALM, но отсутствуют в LIQUIDITY**

```sql
-- Запрос на con_id, которые есть в ALM, но отсутствуют в LIQUIDITY
SELECT br.con_id, br.out_rub
FROM [ALM].[ALM].[VW_Balance_Rest_All] br
WHERE br.dt_rep = '2025-02-28'
  AND br.dt_open BETWEEN '2025-02-01' AND '2025-02-28'
  AND br.section_name = N'Срочные'
  AND br.block_name = N'Привлечение ФЛ'
  AND br.od_flag = 1
  AND br.out_rub IS NOT NULL
  AND br.cur = '810'
  AND br.is_floatrate = 0
  AND br.TSEGMENTNAME = N'Розничный бизнес'
  AND br.con_id NOT IN (
      SELECT dc.con_id
      FROM LIQUIDITY.liq.DepositContract_all dc WITH (NOLOCK)
      WHERE dc.DT_OPEN >= '2025-02-01' AND dc.DT_OPEN < '2025-03-01'
        AND dc.CLI_SUBTYPE = 'INDIV'
        AND dc.PROD_NAME  <> 'Эскроу'
        AND dc.CONVENTION <> 'ON_DEMAND'
        AND dc.DT_CLOSE_PLAN <> '4444-01-01'
        AND dc.seg_name <> 'ДЧБО'
  );
```

---

## **2. Сравнение по месяцам (декабрь 2024 — май 2025)**

```sql
-- Общая таблица по месяцам: count и сумма по обоим источникам
WITH month_list AS (
    SELECT CAST('2024-12-01' AS DATE) AS month_start
    UNION ALL
    SELECT DATEADD(MONTH, 1, month_start)
    FROM month_list
    WHERE month_start < '2025-05-01'
),
ALM_data AS (
    SELECT
        ml.month_start,
        COUNT(br.con_id) AS alm_count,
        SUM(br.out_rub) AS alm_out_rub
    FROM month_list ml
    LEFT JOIN [ALM].[ALM].[VW_Balance_Rest_All] br
        ON br.dt_open >= ml.month_start
        AND br.dt_open < DATEADD(MONTH, 1, ml.month_start)
        AND br.section_name = N'Срочные'
        AND br.block_name = N'Привлечение ФЛ'
        AND br.od_flag = 1
        AND br.out_rub IS NOT NULL
        AND br.cur = '810'
        AND br.is_floatrate = 0
        AND br.TSEGMENTNAME = N'Розничный бизнес'
        AND br.dt_rep = EOMONTH(ml.month_start)
    GROUP BY ml.month_start
),
LIQ_data AS (
    SELECT
        ml.month_start,
        COUNT(dc.con_id) AS liq_count,
        SUM(dc.balance_rub) AS liq_balance_rub
    FROM month_list ml
    LEFT JOIN LIQUIDITY.liq.DepositContract_all dc WITH (NOLOCK)
        ON dc.DT_OPEN >= ml.month_start
        AND dc.DT_OPEN < DATEADD(MONTH, 1, ml.month_start)
        AND dc.CLI_SUBTYPE = 'INDIV'
        AND dc.PROD_NAME  <> 'Эскроу'
        AND dc.CONVENTION <> 'ON_DEMAND'
        AND dc.DT_CLOSE_PLAN <> '4444-01-01'
        AND dc.seg_name <> 'ДЧБО'
    GROUP BY ml.month_start
)
SELECT 
    alm.month_start,
    alm.alm_count,
    alm.alm_out_rub,
    liq.liq_count,
    liq.liq_balance_rub
FROM ALM_data alm
JOIN LIQ_data liq ON alm.month_start = liq.month_start
ORDER BY alm.month_start;
```

---

Если нужно — добавим сравнение (`delta_count`, `delta_sum`) или выведем в Excel-совместимом виде.
