Ниже — обновлённые скрипты ② и ③ с учётом новых требований:

* добавлен `COUNT(DISTINCT con_id)` — количество контрактов;
* сегментация и группировка расширены полем `t.TSEGMENTNAME`;
* из подзапроса `transfers` убран фильтр `cli_biz <> N'ДЧБО'`;
* из основного фильтра убран `t.TSEGMENTNAME <> N'ДЧБО'`;
* в запросе ③ добавлена группировка по `PROD_NAME_RES`.

---

### ② Агрегаты: объём, средневзвешенные ставки, количество договоров

(группировка: `has_big_transfer_flag / TSEGMENTNAME / is_floatrate / PROD_NAME_RES`)

```sql
/* --- CTE с базовыми данными --- */
;WITH base AS (
    SELECT
        CASE WHEN tr.cli_id IS NULL THEN 0 ELSE 1 END         AS has_big_transfer_flag,
        t.TSEGMENTNAME,
        t.is_floatrate,
        t.PROD_NAME_RES,
        t.out_rub,
        ISNULL(r.rate, t.rate_con) AS rate_con,
        t.rate_trf,
        t.con_id
    FROM   alm.[ALM].[vw_balance_rest_all]               t
    LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate       r
           ON  r.con_id = t.con_id
           AND (CASE WHEN t.dt_open = t.dt_rep
                     THEN DATEADD(DAY,1,t.dt_rep)
                     ELSE t.dt_rep
                END) BETWEEN r.dt_from AND r.dt_to
    LEFT   JOIN (                                       -- клиенты с крупными переводами
        SELECT DISTINCT cli_id
        FROM   ALM.[ehd].[VW_transfers_FL_det]
        WHERE  dt_rep BETWEEN '2025-06-01' AND '2025-06-30'
          AND  direction_type      = N'Переводы себе'
          AND  [целевая категория] = '1'
          AND  transit_max_id IS NULL
          AND  spec_cat NOT IN ('BR','FU','SR')
          AND  amount_net <= -100000
    ) tr ON tr.cli_id = t.cli_id
    WHERE  t.dt_rep       = '2025-06-30'
      AND  t.section_name = N'Накопительный счёт'
      AND  t.block_name   = N'Привлечение ФЛ'
      AND  t.od_flag      = 1
      AND  t.out_rub      IS NOT NULL
      AND  t.cur          = '810'
)
/* --- Агрегирование --- */
SELECT
    has_big_transfer_flag,
    TSEGMENTNAME,
    is_floatrate,
    PROD_NAME_RES,
    SUM(out_rub)                                              AS total_balance_rub,
    COUNT(DISTINCT con_id)                                    AS contracts_cnt,
    SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)          AS w_avg_client_rate,
    SUM(out_rub * rate_trf) / NULLIF(SUM(out_rub),0)          AS w_avg_transfer_rate
FROM   base
GROUP BY
    has_big_transfer_flag,
    TSEGMENTNAME,
    is_floatrate,
    PROD_NAME_RES
ORDER BY
    has_big_transfer_flag,
    TSEGMENTNAME,
    is_floatrate,
    PROD_NAME_RES;
```

---

### ③ Агрегаты по **самой клиентской ставке**

(группировка: `has_big_transfer_flag / TSEGMENTNAME / is_floatrate / PROD_NAME_RES / client_rate`)

```sql
;WITH base AS (   -- тот же CTE, что выше
    SELECT
        CASE WHEN tr.cli_id IS NULL THEN 0 ELSE 1 END AS has_big_transfer_flag,
        t.TSEGMENTNAME,
        t.is_floatrate,
        t.PROD_NAME_RES,
        t.out_rub,
        ISNULL(r.rate, t.rate_con) AS rate_con,
        t.rate_trf,
        t.con_id
    FROM   alm.[ALM].[vw_balance_rest_all]               t
    LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate       r
           ON  r.con_id = t.con_id
           AND (CASE WHEN t.dt_open = t.dt_rep
                     THEN DATEADD(DAY,1,t.dt_rep)
                     ELSE t.dt_rep
                END) BETWEEN r.dt_from AND r.dt_to
    LEFT   JOIN (
        SELECT DISTINCT cli_id
        FROM   ALM.[ehd].[VW_transfers_FL_det]
        WHERE  dt_rep BETWEEN '2025-06-01' AND '2025-06-30'
          AND  direction_type      = N'Переводы себе'
          AND  [целевая категория] = '1'
          AND  transit_max_id IS NULL
          AND  spec_cat NOT IN ('BR','FU','SR')
          AND  amount_net <= -100000
    ) tr ON tr.cli_id = t.cli_id
    WHERE  t.dt_rep       = '2025-06-30'
      AND  t.section_name = N'Накопительный счёт'
      AND  t.block_name   = N'Привлечение ФЛ'
      AND  t.od_flag      = 1
      AND  t.out_rub      IS NOT NULL
      AND  t.cur          = '810'
)
/* --- Агрегирование по конкретной клиентской ставке --- */
SELECT
    has_big_transfer_flag,
    TSEGMENTNAME,
    is_floatrate,
    PROD_NAME_RES,
    CAST(rate_con AS DECIMAL(10,4))               AS client_rate,
    SUM(out_rub)                                  AS total_balance_rub,
    COUNT(DISTINCT con_id)                        AS contracts_cnt,
    SUM(out_rub * rate_trf) / NULLIF(SUM(out_rub),0) AS w_avg_transfer_rate
FROM   base
GROUP BY
    has_big_transfer_flag,
    TSEGMENTNAME,
    is_floatrate,
    PROD_NAME_RES,
    CAST(rate_con AS DECIMAL(10,4))
ORDER BY
    has_big_transfer_flag,
    TSEGMENTNAME,
    is_floatrate,
    PROD_NAME_RES,
    client_rate;
```

> *Если нужны “коробки” по ставкам (например, `<5 %`, `5–7 %`, `>7 %`), замените `CAST(rate_con …)` в блоке `SELECT` и `GROUP BY` на выражение-CASE, как показывал ранее.*

Проверьте, что:

* в представлении `vw_balance_rest_all` действительно есть поле `TSEGMENTNAME`;
* `con_id` уникален для контракта – тогда `COUNT(DISTINCT)` корректно посчитает количество договоров.

Если понадобится что-то ещё уточнить — пишите!
