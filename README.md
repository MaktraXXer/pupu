Ок, больше не «стреляем» по каждой строке ‒ теперь сначала суммируем все переводы клиента за день по всем банкам, а уже потом проверяем порог –100 000 ₽.
Ниже два обнов-скрипта (② и ③). Единственное, что поменялось — логика CTE `transfers_agg`; остальная группировка (TSEGMENTNAME, ставки, COUNT DISTINCT con\_id и т.д.) осталась как в прошлой версии.

---

## ② Сводка по остаткам: объём, средневзвешенные ставки, число договоров

```sql
/* --- CTE: клиенты, у кого ≥100 тыс. ₽ вывода за ЛЮБОЙ день июня --- */
;WITH transfers_agg AS (
     SELECT DISTINCT cli_id
     FROM (
         SELECT
             cli_id,
             dt_rep,
             SUM(amount_net) AS sum_net              -- суммируем по банкам
         FROM ALM.[ehd].[VW_transfers_FL_det]
         WHERE dt_rep BETWEEN '2025-06-01' AND '2025-06-30'
           AND direction_type      = N'Переводы себе'
           AND [целевая категория] = '1'
           AND transit_max_id IS NULL
           AND spec_cat NOT IN ('BR','FU','SR')
         GROUP BY cli_id, dt_rep
         HAVING SUM(amount_net) <= -100000           -- порог уже ПОСЛЕ суммирования
     ) t
),
base AS (  -- остатки на 30.06.2025 + флаг крупных переводов
    SELECT
        CASE WHEN tr.cli_id IS NULL THEN 0 ELSE 1 END AS has_big_transfer_flag,
        t.TSEGMENTNAME,
        t.is_floatrate,
        t.PROD_NAME_RES,
        t.out_rub,
        ISNULL(r.rate, t.rate_con) AS rate_con,
        t.rate_trf,
        t.con_id
    FROM  alm.[ALM].[vw_balance_rest_all]            t
    LEFT  JOIN LIQUIDITY.liq.DepositContract_Rate    r
           ON  r.con_id = t.con_id
           AND (CASE WHEN t.dt_open = t.dt_rep
                     THEN DATEADD(day,1,t.dt_rep)
                     ELSE t.dt_rep
                END) BETWEEN r.dt_from AND r.dt_to
    LEFT  JOIN transfers_agg tr ON tr.cli_id = t.cli_id
    WHERE t.dt_rep       = '2025-06-30'
      AND t.section_name = N'Накопительный счёт'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.out_rub      IS NOT NULL
      AND t.cur          = '810'
)
/* --- итоговый срез --- */
SELECT
    has_big_transfer_flag,
    TSEGMENTNAME,
    is_floatrate,
    PROD_NAME_RES,
    SUM(out_rub)                                              AS total_balance_rub,
    COUNT(DISTINCT con_id)                                    AS contracts_cnt,
    SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)          AS w_avg_client_rate,
    SUM(out_rub * rate_trf) / NULLIF(SUM(out_rub),0)          AS w_avg_transfer_rate
FROM base
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

## ③ Та же сводка, но по **каждой клиентской ставке**

```sql
;WITH transfers_agg AS (     /* -- тот же CTE, см. выше -- */ 
     SELECT DISTINCT cli_id
     FROM (
         SELECT cli_id, dt_rep,
                SUM(amount_net) AS sum_net
         FROM ALM.[ehd].[VW_transfers_FL_det]
         WHERE dt_rep BETWEEN '2025-06-01' AND '2025-06-30'
           AND direction_type      = N'Переводы себе'
           AND [целевая категория] = '1'
           AND transit_max_id IS NULL
           AND spec_cat NOT IN ('BR','FU','SR')
         GROUP BY cli_id, dt_rep
         HAVING SUM(amount_net) <= -100000
     ) t
),
base AS (
    SELECT
        CASE WHEN tr.cli_id IS NULL THEN 0 ELSE 1 END AS has_big_transfer_flag,
        t.TSEGMENTNAME,
        t.is_floatrate,
        t.PROD_NAME_RES,
        t.out_rub,
        ISNULL(r.rate, t.rate_con) AS rate_con,
        t.rate_trf,
        t.con_id
    FROM  alm.[ALM].[vw_balance_rest_all]            t
    LEFT  JOIN LIQUIDITY.liq.DepositContract_Rate    r
           ON  r.con_id = t.con_id
           AND (CASE WHEN t.dt_open = t.dt_rep
                     THEN DATEADD(day,1,t.dt_rep)
                     ELSE t.dt_rep
                END) BETWEEN r.dt_from AND r.dt_to
    LEFT  JOIN transfers_agg tr ON tr.cli_id = t.cli_id
    WHERE t.dt_rep       = '2025-06-30'
      AND t.section_name = N'Накопительный счёт'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.out_rub      IS NOT NULL
      AND t.cur          = '810'
)
SELECT
    has_big_transfer_flag,
    TSEGMENTNAME,
    is_floatrate,
    PROD_NAME_RES,
    CAST(rate_con AS DECIMAL(10,4))               AS client_rate,
    SUM(out_rub)                                  AS total_balance_rub,
    COUNT(DISTINCT con_id)                        AS contracts_cnt,
    SUM(out_rub * rate_trf) / NULLIF(SUM(out_rub),0) AS w_avg_transfer_rate
FROM base
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

### Проверить

* Поле **amount\_net** в `VW_transfers_FL_det` отрицательно при выводе (как раньше).
* `cli_id` в обоих представлениях совпадает по формату.
* В `vw_balance_rest_all` есть `TSEGMENTNAME`; `con_id` — уникальный номер договора.

Если ещё что-то всплывёт — сориентируй, подправим.
