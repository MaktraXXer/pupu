```sql
/* ----------------------------------------------
   Подробная выгрузка ДВС-портфеля и «новых»  
   за период 18 марта 2025 – 30 марта 2025
   (все поля из alm.ALM.vw_balance_rest_all
   + ставка из LIQUIDITY.liq.DepositContract_Rate)
------------------------------------------------*/
SELECT
    t.* ,                            -- все колонки баланса
    r.rate AS rate_liq               -- ставка из liq.DepositContract_Rate
FROM alm.ALM.vw_balance_rest_all AS t
/* ------------------------------------------------
   Ставку берём по той же логике, что и в процедуре:
   – если дата открытия = дате среза, то смотрим
     «следующий день»; иначе саму dt_rep
--------------------------------------------------*/
LEFT JOIN LIQUIDITY.liq.DepositContract_Rate AS r
       ON r.con_id = t.con_id
      AND CASE
              WHEN t.dt_open = t.dt_rep
                   THEN DATEADD(day, 1, t.dt_rep)
              ELSE t.dt_rep
          END BETWEEN r.dt_from AND r.dt_to
/* ----------------------------------------------
   Фильтры, совпадающие с процедурой
----------------------------------------------*/
WHERE t.dt_rep BETWEEN '2025-03-18' AND '2025-03-30'     -- включительно  
  AND t.section_name = N'До востребования'               -- ДВС  
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'                             -- рубли
/* ----------------------------------------------*/
ORDER BY
    t.con_id,
    t.dt_rep;
```

Запрос выводит полный набор колонок из `alm.ALM.vw_balance_rest_all` за указанный период, добавляет соответствующую ставку из `LIQUIDITY.liq.DepositContract_Rate` и отсортирован по `con_id, dt_rep`, что удобно для проверки качества данных по портфелю и «новым» договорам в ДВС.


Ниже два запроса-«детектора».

---

### 1. Найти «подозрительные» con\_id

(договор впервые попал в срез ДВС **как «новый»** между 18 и 30 марта 2025, а в марте 2025 имеет записи с другим сегментом — «Накопительный счёт»).

```sql
/* ---------- Параметры периода ---------- */
DECLARE @dvs_from date =  N'2025-03-18';
DECLARE @dvs_to   date =  N'2025-03-30';
DECLARE @march_from date = N'2025-03-01';
DECLARE @march_to   date = N'2025-03-31';

/* ---------- 1. con_id, попавшие в ДВС как «новые» ---------- */
WITH dvs_new AS (
    SELECT DISTINCT con_id
    FROM alm.ALM.vw_balance_rest_all
    WHERE dt_rep BETWEEN @dvs_from AND @dvs_to
      AND section_name = N'До востребования'
      AND dt_open      = dt_rep              -- «новый»
      AND block_name   = N'Привлечение ФЛ'
      AND od_flag      = 1
      AND cur          = '810'
),

/* ---------- 2. все мартовские записи по найденным con_id ---------- */
march_data AS (
    SELECT  t.*,
            LAG(t.section_name) OVER (PARTITION BY t.con_id ORDER BY t.dt_rep) AS prev_section
    FROM alm.ALM.vw_balance_rest_all AS t
    JOIN dvs_new d  ON d.con_id = t.con_id
    WHERE t.dt_rep BETWEEN @march_from AND @march_to
      AND t.block_name = N'Привлечение ФЛ'
      AND t.od_flag    = 1
      AND t.cur        = '810'
),

/* ---------- 3. моменты, когда сегмент сменился ---------- */
seg_change AS (
    SELECT *
    FROM march_data
    WHERE prev_section IS NOT NULL
      AND prev_section <> section_name
)

SELECT DISTINCT con_id              -- список «подозрительных» договоров
FROM seg_change
ORDER BY con_id;
```

---

### 2. Показать **конкретные примеры** смены сегмента

(можно дописать `TOP (100)` или фильтр по нужному `con_id`).

```sql
/* продол­жаем CTE seg_change из примера 1
   (или сделайте WITH … seg_change AS (…) как выше) */
SELECT
    con_id,
    dt_rep,
    prev_section     AS section_prev,
    section_name     AS section_curr,
    out_rub,
    rate_con         AS rate_balance
FROM seg_change
ORDER BY con_id, dt_rep;
```

**Что делает проверка**

1. Ищет договора, которые **впервые** появились в ДВС-портфеле как «новые» ( `dt_open = dt_rep`) в проблемном окне 18–30 марта.
2. Берёт все их записи за март 2025 и с помощью `LAG` находит даты, когда `section_name` меняется.
3. Возвращает список таких con\_id (запрос 1) и сами строки-«переключатели» (запрос 2) — так вы сможете убедиться, что один и тот же договор попеременно идёт то в «До востребования», то в «Накопительный счёт».
