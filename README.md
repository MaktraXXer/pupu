Ниже ― готовый «черновик» решения, которым можно сразу же пользоваться в тестовой базе **ALM\_TEST**.
Я сохранил всю логику исходной витрины, но:

* **добавил** подтягивание «эталонных» ставок из `LIQUIDITY.liq.DepositContract_Rate` (п. 1);
* **дал возможность выбирать любую отчётную дату** `@rep_dt` (п. 2).

  > В обычном представлении параметры недоступны, поэтому я перевёл витрину в *inline* табличную функцию — одной строкой она же превращается обратно в представление по-умолчанию.
* **создал отдельную схему** `urep_ex` в базе **ALM\_TEST** (п. 3).
* если «эталонной» ставки нет, расчёт идёт по старому полю `t.rate_con` (п. 4).

---

## 1. Скрипт развёртывания в ALM\_TEST

```sql
/* ─────────── 0. Подготовка схемы ─────────── */
USE ALM_TEST;
GO
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'urep_ex')
    EXEC('CREATE SCHEMA urep_ex AUTHORIZATION dbo;');
GO

/* ─────────── 1. Функция с параметром отчётной даты ─────────── */
CREATE OR ALTER FUNCTION urep_ex.fn_report_FL_NewKopilki
(
    @rep_dt DATE = NULL        -- если NULL → берём max(dt_rep)
)
RETURNS TABLE
AS
RETURN
/* 1.1 – фиксируем целевую rep-дату */
WITH rep_dt AS (
    SELECT COALESCE(@rep_dt,
                    (SELECT MAX(dt_rep) FROM ALM.balance_rest_all_AGG)) AS dt
),
/* 1.2 – базовые строки из vw_balance_rest_all + календарь */
base AS (
    SELECT
        t.SEGMENT_ID,
        c1.date                    AS opn_dt_start,
        c2.date                    AS opn_dt_end,
        rdt.dt                     AS rep_dt,
        t.con_id,
        t.DT_OPEN,
        t.OUT_RUB,
        t.rate_con,                -- старая ставка
        t.rate_trf
    FROM  alm.info.VW_calendar            AS c1             -- начало окна
    CROSS JOIN alm.info.VW_calendar        AS c2             -- конец окна
    CROSS JOIN rep_dt                      AS rdt            -- выбранная rep-дата
    JOIN  alm.ALM.vw_balance_rest_all      AS t
          ON  t.dt_rep  = rdt.dt
          AND t.dt_open BETWEEN c1.date AND c2.date
          AND t.acc_role        = N'LIAB'
          AND t.block_name      = N'Привлечение ФЛ'
          AND t.section_name    = N'Накопительный счёт'
          AND t.od_flag         = 1
          AND t.OUT_RUB IS NOT NULL
          AND ISNULL(t.is_floatrate,0) <> 1
    WHERE c1.date BETWEEN '2020-01-01' AND GETDATE()
)
/* 1.3 – подцепляем «правильные» ставки */
, joined AS (
    SELECT  b.*,
            r.rate AS rate_ref               -- эталон (можно заменить на нужное поле)
    FROM    base AS b
    /* LEFT JOIN – чтобы не терять строки, когда ставки нет */
    LEFT JOIN LIQUIDITY.liq.DepositContract_Rate AS r
           ON  r.con_id = b.con_id
           /* «+1 день» для новых; иначе – сама rep-дата */
           AND CASE WHEN b.DT_OPEN = b.rep_dt
                    THEN DATEADD(DAY,1,b.rep_dt)
                    ELSE b.rep_dt END
               BETWEEN r.dt_from AND r.dt_to
)
/* 1.4 – агрегирование ровно как в исходной витрине + новые метрики */
SELECT
    SEGMENT_ID,
    SUM(OUT_RUB)/1e6                                       AS SUM_OUT_RUB_mln,
    /* исходная средневзвешенная ставка */
    SUM(OUT_RUB * rate_con)/SUM(OUT_RUB)                   AS conrate_SRVZ_orig,
    /* ставка по тарифу (как была) */
    SUM(OUT_RUB * rate_trf)/SUM(OUT_RUB)                   AS trf_SRVZ,
    /* средневзвешенная по эталонным (или fallback к старой) */
    SUM(OUT_RUB * COALESCE(rate_ref, rate_con))
        / SUM(OUT_RUB)                                     AS conrate_SRVZ_ref,
    rep_dt,
    opn_dt_start,
    opn_dt_end,
    COUNT(*)                                               AS c,
    /* полезная статистика: сколько строк без эталона */
    SUM(CASE WHEN rate_ref IS NULL THEN 1 ELSE 0 END)      AS cnt_no_rate
FROM joined
GROUP BY SEGMENT_ID, opn_dt_start, opn_dt_end, rep_dt;
GO

/* ─────────── 2. Представление по-умолчанию (для «последней» даты) ─────────── */
CREATE OR ALTER VIEW urep_ex.VW_report_FL_NewKopilki_enriched
AS
SELECT * FROM urep_ex.fn_report_FL_NewKopilki(NULL);
GO
```

### Как использовать

```sql
/* 1) как и раньше – «последний» отчёт */
SELECT *
FROM ALM_TEST.urep_ex.VW_report_FL_NewKopilki_enriched
WHERE opn_dt_start = '2025-05-01'
  AND opn_dt_end   = '2025-06-08';

/* 2) с конкретной датой отчёта */
SELECT *
FROM ALM_TEST.urep_ex.fn_report_FL_NewKopilki('2025-06-23')
WHERE opn_dt_start = '2025-05-01'
  AND opn_dt_end   = '2025-06-08';
```

---

## 2. Логика джойна ставок ― пошагово

| Шаг | Условие                                                                                                         | Зачем                                                                                                |
| --- | --------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| 1.  | **`r.con_id = t.con_id`**                                                                                       | строгая связка «контракт–ставка»                                                                     |
| 2.  | Вычисляем «момент проверки»:<br>`CASE WHEN t.dt_open = t.dt_rep THEN DATEADD(day,1,t.dt_rep) ELSE t.dt_rep END` | для контрактов, открытых в отчётный день, ставка начинает действовать со следующего календарного дня |
| 3.  | Попадаем в интервал действия ставки:<br>`... BETWEEN r.dt_from AND r.dt_to`                                     | чтобы взять именно ту ставку, которая действовала в этот момент                                      |
| 4.  | `LEFT JOIN`                                                                                                     | если интервала нет → поле `r.rate` = NULL, а в расчёте используется `t.rate_con` через `COALESCE`    |

Таким образом:

```sql
effective_rate = COALESCE(r.rate, t.rate_con)
```

*Если в `DepositContract_Rate` хранится несколько разных полей (номинальная / эффективная и т.д.) — достаточно поменять `r.rate` на нужный столбец, остальная логика не меняется.*

---

## 3. Что ещё можно быстро посмотреть

```sql
/* распределение отклонений, б.п. */
SELECT
    SEGMENT_ID,
    AVG(diff_bps)  AS avg_diff,
    MAX(diff_bps)  AS max_diff,
    COUNT(*)       AS rows_cnt
FROM (
    SELECT 
        SEGMENT_ID,
        (COALESCE(rate_ref, rate_con) - rate_con) * 10000 AS diff_bps
    FROM ALM_TEST.urep_ex.fn_report_FL_NewKopilki('2025-06-23')
) d
GROUP BY SEGMENT_ID
ORDER BY ABS(avg_diff) DESC;
```

Это сразу покажет, в каких сегментах «новые» ставки сильнее всего расходятся со старыми.

---

### Готово!

Скрипт можно накатывать в тестовой среде; если всё ок, достаточно переместить функцию/представление в прод-базу **ALM** (или просто изменить префикс базы). Будут вопросы — пишите, скорректируем.
