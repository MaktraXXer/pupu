```sql
/* -----------------------------------------------------------
   Плавающие накопительные счёты ФЛ,
   открытые 30-июн-2025 или 01-июл-2025,
   по срезу баланса на 30-июн-2025.

   Показываем сразу две ставки:
     • rate_con  – «балансовая» (из vw_balance_rest_all);
     • rate_ref  – «эталонная»   (из DepositContract_Rate).
----------------------------------------------------------- */
DECLARE @rep_dt  date = '20250630';   -- отчётная дата = баланс на 30 июня 2025

SELECT
    t.con_id,
    t.dt_open,
    t.OUT_RUB                       AS bal_out_rub,
    t.rate_con                      AS rate_con_src,   -- ставка из баланса
    r.rate                          AS rate_ref_src,   -- ставка из справочника
    t.is_floatrate                  -- всегда 1 (плавающая)
FROM  ALM.ALM.vw_balance_rest_all      AS t
LEFT  JOIN LIQUIDITY.liq.DepositContract_Rate AS r
       ON  r.con_id = t.con_id
       /* та же логика, что в витрине: для «день-в-день» берём rep_dt+1 */
       AND CASE WHEN t.dt_open = @rep_dt
                THEN DATEADD(day,1,@rep_dt)
                ELSE @rep_dt END
           BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep       = @rep_dt                   -- баланс на 30-июн-2025
  AND  t.acc_role     = N'LIAB'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.section_name = N'Накопительный счёт'
  AND  t.od_flag      = 1
  AND  ISNULL(t.is_floatrate,0) = 1               -- только плавающие НС
  AND  t.dt_open IN ('20250630','20250701')       -- открыты 30.06 или 01.07
ORDER BY t.dt_open, t.con_id;
```

### Что делает запрос

| Шаг                                  | Логика                                                                                                                                                                            |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@rep_dt = '20250630'`               | фиксируем отчётную дату (баланс на 30 июня 2025).                                                                                                                                 |
| Фильтр по `is_floatrate = 1`         | берём **только плавающие** накопительные счета.                                                                                                                                   |
| `dt_open IN ('20250630','20250701')` | оставляем договоры, открытые 30 июня или 1 июля.                                                                                                                                  |
| `rate_con_src`                       | ставка, записанная в витрине баланса (`vw_balance_rest_all`).                                                                                                                     |
| `rate_ref_src`                       | эталонная ставка из `DepositContract_Rate`; выбирается по тому же алгоритму, что и в вашей витрине (для договоров, открытых ровно в отчётный день, берём интервал «rep\_dt + 1»). |

> ⚠ **Замечание**
> В срезе на 30 июня договоры, открытые 1 июля, могут отсутствовать (это зависит от логики загрузки баланса). Если строк не окажется, значит такие счета попадают только в баланс на 01.07.2025 — тогда запустите тот же запрос с `@rep_dt = '20250701'`.

---

### Если нужна агрегированная сводка

```sql
SELECT
    t.is_floatrate,
    AVG(t.rate_con)       AS avg_rate_con,
    AVG(r.rate)           AS avg_rate_ref,
    SUM(t.OUT_RUB)/1e6    AS sum_out_rub_mln,
    COUNT(*)              AS cnt_contracts
FROM  ALM.ALM.vw_balance_rest_all AS t
LEFT  JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON r.con_id = t.con_id
       AND CASE WHEN t.dt_open = @rep_dt
                THEN DATEADD(day,1,@rep_dt)
                ELSE @rep_dt END
           BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep = @rep_dt
  AND  t.acc_role     = N'LIAB'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.section_name = N'Накопительный счёт'
  AND  t.od_flag      = 1
  AND  ISNULL(t.is_floatrate,0) = 1
  AND  t.dt_open IN ('20250630','20250701')
GROUP BY t.is_floatrate;
```

Эта форма покажет средневзвешенные ставки (`AVG`, при равных объёмах) и суммарный остаток, сгруппировав всё равно по `is_floatrate` (будет один блок со значением 1).
