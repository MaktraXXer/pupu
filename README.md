Ниже две готовых заготовки SQL (копируй-вставляй).

1. “Generation” — винтаж по месяцу
Клиенту навсегда присваиваем метку generation вида 2024-01, 2024-02 … — месяц, когда у него впервые появился один из вкладов из списка.
В остальные месяцы его данные остаются, но с той же меткой.

sql
Копировать
Редактировать
/* ===== 0. Целевые вклады ===== */
DECLARE @Products TABLE (prod_name_res NVARCHAR(255) PRIMARY KEY);
INSERT INTO @Products VALUES
('Надёжный'), ('Надёжный VIP'),
('Надёжный премиум'), ('Надёжный промо'),
('Надёжный старт');

/* ===== 1. Первая встреча клиента с нужным вкладом ===== */
WITH first_touch AS (
    SELECT
        cli_id,
        MIN(dt_Rep) AS first_dt_rep
    FROM  ALM.ALM.Balance_Rest_All WITH (NOLOCK)
    WHERE prod_name_res IN (SELECT prod_name_res FROM @Products)
      AND dt_Rep IN ( '2024-01-31','2024-02-29','2024-03-31','2024-04-30',
                      '2024-05-31','2024-06-30','2024-07-31','2024-08-31',
                      '2024-09-30','2024-10-31','2024-11-30','2024-12-31',
                      '2025-01-31','2025-02-28','2025-03-31','2025-04-30',
                      '2025-05-20')
      AND section_name  IN ('Срочные','До востребования','Накопительный счёт')
      AND cur           = '810'
      AND od_flag       = 1
      AND is_floatrate  = 0
      AND ACC_ROLE      = 'LIAB'
      AND AP            = 'Пассив'
      AND TSEGMENTNAME IN ('ДЧБО','Розничный бизнес')
      AND BLOCK_NAME    = 'Привлечение ФЛ'
      AND OUT_RUB IS NOT NULL
    GROUP BY cli_id
)
, generation AS (
    SELECT
        cli_id,
        FORMAT(first_dt_rep,'yyyy-MM') AS generation   -- «2025-02», «2024-11» и т.д.
    FROM first_touch
)

/* ===== 2. Все балансы клиентов, помеченные их generation ===== */
, cte AS (
    SELECT
        bra.dt_Rep,
        g.generation,
        bra.SECTION_NAME,
        bra.TSEGMENTNAME,
        bra.PROD_NAME_RES,
        bra.con_id,
        bra.OUT_RUB
    FROM  ALM.ALM.Balance_Rest_All bra WITH (NOLOCK)
    JOIN  generation g  ON g.cli_id = bra.cli_id
    WHERE bra.dt_Rep IN ( '2024-01-31','2024-02-29','2024-03-31','2024-04-30',
                          '2024-05-31','2024-06-30','2024-07-31','2024-08-31',
                          '2024-09-30','2024-10-31','2024-11-30','2024-12-31',
                          '2025-01-31','2025-02-28','2025-03-31','2025-04-30',
                          '2025-05-20')
      AND bra.section_name  IN ('Срочные','До востребования','Накопительный счёт')
      AND bra.cur           = '810'
      AND bra.od_flag       = 1
      AND bra.is_floatrate  = 0
      AND bra.ACC_ROLE      = 'LIAB'
      AND bra.AP            = 'Пассив'
      AND bra.TSEGMENTNAME IN ('ДЧБО','Розничный бизнес')
      AND bra.BLOCK_NAME    = 'Привлечение ФЛ'
      AND bra.OUT_RUB IS NOT NULL
)

/* ===== 3. Агрегация с меткой generation ===== */
SELECT
    dt_Rep,
    generation,
    SECTION_NAME,
    TSEGMENTNAME,
    PROD_NAME_RES,
    SUM(OUT_RUB)           AS sum_OUT_RUB,
    COUNT(DISTINCT con_id) AS count_CON_ID
FROM cte
GROUP BY
    dt_Rep, generation, SECTION_NAME, TSEGMENTNAME, PROD_NAME_RES
ORDER BY
    dt_Rep, SECTION_NAME, TSEGMENTNAME, PROD_NAME_RES;
Что получаем

Любая строка всегда несёт generation — месяц первого вклада клиента.

Фильтруя WHERE generation = '2025-02', увидите только «февральских новичков» даже в последующие месяцы (если не ограничите dt_Rep).

2. Сортировка объёма по продукту за февраль–май 2025
sql
Копировать
Редактировать
/* суммарные рублёвые пассивы по каждому продукту
   за 28-Фев, 31-Мар, 30-Апр, 20-Мая 2025 */
SELECT
    PROD_NAME_RES,
    SUM(OUT_RUB) AS sum_OUT_RUB
FROM ALM.ALM.Balance_Rest_All WITH (NOLOCK)
WHERE dt_Rep IN ('2025-02-28','2025-03-31','2025-04-30','2025-05-20')
  AND section_name  IN ('Срочные','До востребования','Накопительный счёт')
  AND cur           = '810'
  AND od_flag       = 1
  AND is_floatrate  = 0
  AND ACC_ROLE      = 'LIAB'
  AND AP            = 'Пассив'
  AND TSEGMENTNAME IN ('ДЧБО','Розничный бизнес')
  AND BLOCK_NAME    = 'Привлечение ФЛ'
  AND OUT_RUB IS NOT NULL
GROUP BY PROD_NAME_RES
ORDER BY sum_OUT_RUB DESC;      -- сортировка по убыванию объёма
Если нужно оставить только те продукты из @Products, просто добавьте AND prod_name_res IN (SELECT prod_name_res FROM @Products).

Проверь, что и “generation”-запрос, и сортировка работают как надо; если где-то ещё надо подкрутить — пиши!

Вот обновлённый запрос — к суммарному объёму по каждому продукту добавлен средневзвешенный `con_rate`, рассчитанный по весу `OUT_RUB` (если ставка хранится в поле с другим именем — замени его в двух местах).

```sql
/* -----------------------------------------------
   Сумма рублёвых пассивов и средневзвешенная ставка
   за февраль-май 2025 по каждому PROD_NAME_RES
----------------------------------------------- */
SELECT
    PROD_NAME_RES,

    /* общий объём */
    SUM(OUT_RUB)                                        AS sum_OUT_RUB,

    /* средневзвешенная ставка = Σ(rate * объём) / Σ(объём) */
    SUM(OUT_RUB * con_rate) / NULLIF(SUM(OUT_RUB), 0.0) AS wavg_con_rate

FROM ALM.ALM.Balance_Rest_All WITH (NOLOCK)

WHERE dt_Rep IN ('2025-02-28','2025-03-31','2025-04-30','2025-05-20')
  AND section_name  IN ('Срочные','До востребования','Накопительный счёт')
  AND cur           = '810'
  AND od_flag       = 1
  AND is_floatrate  = 0
  AND ACC_ROLE      = 'LIAB'
  AND AP            = 'Пассив'
  AND TSEGMENTNAME IN ('ДЧБО','Розничный бизнес')
  AND BLOCK_NAME    = 'Привлечение ФЛ'
  AND OUT_RUB       IS NOT NULL
  AND con_rate      IS NOT NULL          -- чтобы не смешивать NULL-ставки (убери, если нужны)

GROUP BY PROD_NAME_RES
ORDER BY sum_OUT_RUB DESC;               -- сортировка по объёму (можно по wavg_con_rate)
```

### Что важно

* **Формула**: `Σ(rate × объём) / Σ(объём)` — классический средневзвешенный.
* `NULLIF(SUM(OUT_RUB), 0.0)` защищает от деления на ноль, если вдруг объёма нет.
* Если нужно оставить только продукты из вашего списка `@Products`, добавь строку
  `AND prod_name_res IN (SELECT prod_name_res FROM @Products)`
  перед `GROUP BY`.

При необходимости меняй сортировку (`ORDER BY wavg_con_rate DESC`, например) или выводи оба поля в нужном формате.
