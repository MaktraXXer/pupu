Вот вариант, где результаты сразу «раскладываются» отдельно по двум сегментам **ДЧБО** и **Розничный бизнес**.

```sql
/* ========== 1. Остатки по дням до закрытия, разбитые по сегменту ========== */
WITH base AS (
    SELECT
        TSEGMENTNAME,
        CASE
            WHEN DT_CLOSE      = '4444-01-01'
              OR DT_CLOSE_FACT = '4444-01-01'
              OR DT_CLOSE_PLAN = '4444-01-01'
              OR remainingdays IS NULL                    THEN N'бессрочно'   -- нет конечной даты
            WHEN remainingdays < 0                        THEN N'просрочено'  -- срок уже истёк
            ELSE CAST(remainingdays AS varchar(20))
        END                   AS days_left,
        OUT_RUB               AS out_rub
    FROM  ALM.balance_rest_all WITH (NOLOCK)
    WHERE MAP_IS_CASH  = 1
      AND TSEGMENTNAME IN (N'ДЧБО', N'Розничный бизнес')
      AND AP           = N'Пассив'
      AND BLOCK_NAME   = N'Привлечение ФЛ'
      AND SECTION_NAME IN (N'Срочные', N'До востребования', N'Накопительный счёт')
      AND dt_rep       = '20250511'           -- календарные сутки 11-го мая 2025 г.
)

SELECT
    TSEGMENTNAME,
    days_left,
    SUM(out_rub)/1e6  AS total_mln          -- млн руб.
FROM   base
GROUP  BY TSEGMENTNAME, days_left
ORDER  BY
    TSEGMENTNAME,
    CASE
        WHEN days_left = N'просрочено' THEN 999998
        WHEN days_left = N'бессрочно'  THEN 999999
        ELSE TRY_CAST(days_left AS int)      -- 0,1,2,…
    END;
```

```sql
/* ========== 2. Совокупный объём средств, разбитый по сегменту ========== */
SELECT
    TSEGMENTNAME,
    SUM(SOUT_RUB) / 1e6  AS total_mln       -- млн руб.
FROM  ALM.VW_alm_balance_AGG WITH (NOLOCK)
WHERE IS_CASH      = 1
  AND TSEGMENTNAME IN (N'ДЧБО', N'Розничный бизнес')
  AND AP           = N'Пассив'
  AND BLOCK_NAME   = N'Привлечение ФЛ'
  AND SECTION_NAME IN (N'Срочные', N'До востребования', N'Накопительный счёт')
  AND DT_REP       = '20250511'             -- 11-е мая 2025 г.
GROUP BY TSEGMENTNAME
ORDER  BY TSEGMENTNAME;
```

### Что изменилось

1. **TSEGMENTNAME вынесен в SELECT и GROUP BY**, поэтому каждая выборка автоматически делится на две строки – по каждому сегменту.
2. В обеих запросах список `TSEGMENTNAME IN (N'ДЧБО', N'Розничный бизнес')` оставляет только нужные сегменты.
3. Во втором запросе привёл значение `'Срочные '` без лишнего пробела (как в первом запросе) – иначе возможен промах по условию.
4. Для наглядности сразу перевёл суммы в миллионы (`/ 1e6`). Уберите деление, если нужны рубли «как есть».

Эти два скрипта дадут отдельные итоги по каждому сегменту. Если нужна ещё какая-то пост-обработка (например, поворот/сводная таблица), дайте знать – подправлю.
