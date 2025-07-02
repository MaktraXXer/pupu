Ниже — три готовых скрипта.
Я старался оставить все ваши фильтры без изменений; проверьте имена полей (у меня в балансовом представлении предполагается t.cli\_id).

---

### 1)  Детальная выборка + флаг «был ли крупный внутрибанковский перевод»

```sql
/* --- A. список клиентов с «переводами себе» 1-30 июня 2025 г. --- */
;WITH transfers AS (
    SELECT DISTINCT cli_id
    FROM   ALM.[ehd].[VW_transfers_FL_det]
    WHERE  dt_rep BETWEEN '2025-06-01' AND '2025-06-30'
      AND  direction_type      = N'Переводы себе'
      AND  cli_biz            <> N'ДЧБО'
      AND  [целевая категория] = '1'
      AND  transit_max_id IS NULL
      AND  spec_cat NOT IN ('BR','FU','SR')
      AND  amount_net <= -100000          -- ≥ 100 тыс. руб. в минус (вывод)
)
/* --- B. остатки по накопительным счетам на 30.06.2025 --- */
SELECT
    t.dt_rep,
    t.is_floatrate,
    t.PROD_NAME_RES,
    t.dt_open,
    t.out_rub,
    t.con_id,
    ISNULL(r.rate, t.rate_con) AS rate_con,   -- «клиентская» ставка
    t.rate_trf,                               -- трансфертная
    CASE WHEN tr.cli_id IS NULL THEN 0 ELSE 1 END AS has_big_transfer_flag
FROM   alm.[ALM].[vw_balance_rest_all]               t
LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate       r
       ON  r.con_id = t.con_id
       AND (CASE WHEN t.dt_open = t.dt_rep
                 THEN DATEADD(DAY,1,t.dt_rep)  -- «+1 день» для новых
                 ELSE t.dt_rep
            END) BETWEEN r.dt_from AND r.dt_to
LEFT   JOIN transfers tr
       ON  tr.cli_id = t.cli_id                  -- флаг «был перевод»
WHERE  t.dt_rep       = '2025-06-30'
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.out_rub      IS NOT NULL
  AND  t.cur          = '810'
  AND  t.TSEGMENTNAME <> N'ДЧБО';
```

---

### 2)  Агрегаты: объём + ср-взв. клиентская и трансфертная ставка

(группировка: флаг перевода / is\_floatrate / PROD\_NAME\_RES)

```sql
;WITH base AS (  -- то же, что запрос 1, но только нужные поля
    SELECT
        CASE WHEN tr.cli_id IS NULL THEN 0 ELSE 1 END AS has_big_transfer_flag,
        t.is_floatrate,
        t.PROD_NAME_RES,
        t.out_rub,
        ISNULL(r.rate, t.rate_con) AS rate_con,
        t.rate_trf
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
          AND  cli_biz            <> N'ДЧБО'
          AND  [целевая категория] = '1'
          AND  transit_max_id IS NULL
          AND  spec_cat NOT IN ('BR','FU','SR')
          AND  amount_net <= -100000
    ) tr  ON tr.cli_id = t.cli_id
    WHERE  t.dt_rep       = '2025-06-30'
      AND  t.section_name = N'Накопительный счёт'
      AND  t.block_name   = N'Привлечение ФЛ'
      AND  t.od_flag      = 1
      AND  t.out_rub      IS NOT NULL
      AND  t.cur          = '810'
      AND  t.TSEGMENTNAME <> N'ДЧБО'
)
SELECT
    has_big_transfer_flag,
    is_floatrate,
    PROD_NAME_RES,
    SUM(out_rub)                                              AS total_balance_rub,
    SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)          AS w_avg_client_rate,
    SUM(out_rub * rate_trf) / NULLIF(SUM(out_rub),0)          AS w_avg_transfer_rate
FROM   base
GROUP BY
    has_big_transfer_flag,
    is_floatrate,
    PROD_NAME_RES
ORDER BY
    has_big_transfer_flag,
    is_floatrate,
    PROD_NAME_RES;
```

---

### 3)  То же, но группируем по самой клиентской ставке

Если нужно именно по каждой **конкретной** ставке:

```sql
;WITH base AS ( -- тот же CTE, что в запросе 2
    /* …см. выше… */
)
SELECT
    has_big_transfer_flag,
    is_floatrate,
    CAST(rate_con AS DECIMAL(10,4))               AS client_rate,   -- ровно как хранится
    SUM(out_rub)                                  AS total_balance_rub,
    SUM(out_rub * rate_trf) / NULLIF(SUM(out_rub),0) AS w_avg_transfer_rate
FROM   base
GROUP BY
    has_big_transfer_flag,
    is_floatrate,
    CAST(rate_con AS DECIMAL(10,4))
ORDER BY
    has_big_transfer_flag,
    is_floatrate,
    client_rate;
```

> **Если ставки нужно сгруппировать «коробками»** (например «< 5 %», «5-7 %», «> 7 %»), замените `client_rate` на
> `CASE WHEN rate_con < 5 THEN '<5%' WHEN rate_con <= 7 THEN '5-7%' ELSE '>7%' END`.

---

### Что проверить

1. Наличие поля **cli\_id** в `vw_balance_rest_all`.
2. Типы полей `rate_con`, `rate_trf` — если это `varchar`, обязательно приведите к `decimal`.
3. При весовых средних я использовал `out_rub` как вес.

Если нужно что-то доработать — дайте знать!
