Ок, зафиксировал логику:

1. **Сначала** определяем «исчезнувших» по month-end anti-join (это первично).
2. **Потом** из них оставляем только те, у кого дата закрытия попадает в окно **\[19.08.2024; 15.09.2024]**, причём берём `close_dt = COALESCE(dt_close_fact, dt_close)`.
3. Объём выхода — берём из «предыдущего» среза (07-31 для августовских, 08-31 для сентябрьских).
4. Считаем бакеты и долю внутри сегмента. Без лишних полей.

```sql
-- === ОКНО ===
DECLARE @win_start date = '2024-08-19';   -- включительно
DECLARE @win_end_ex date = '2024-09-16';  -- полуинтервал (исключая 16.09, т.е. по 15.09 включительно)

-- === MONTH-END ДАТЫ ===
DECLARE @m7 date = '2024-07-31';
DECLARE @m8 date = '2024-08-31';
DECLARE @m9 date = '2024-09-30';

-- === БАЗОВЫЕ ФИЛЬТРЫ ===
-- Срочные / Привлечение ФЛ / рубли / od_flag=1 / есть out_rubm
WITH m7 AS (
    SELECT t.cli_id, t.con_id, t.TSEGMENTNAME, t.termdays, t.out_rub, t.out_rubm,
           t.dt_open, t.dt_close_fact, t.dt_close AS dt_close_planned
    FROM alm.[ALM].[vw_balance_rest_all] t
    WHERE t.dt_rep = @m7
      AND t.section_name = N'Срочные'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND ISNULL(t.out_rubm,0) > 0
),
m8 AS (
    SELECT t.cli_id, t.con_id, t.TSEGMENTNAME, t.termdays, t.out_rub, t.out_rubm,
           t.dt_open, t.dt_close_fact, t.dt_close AS dt_close_planned
    FROM alm.[ALM].[vw_balance_rest_all] t
    WHERE t.dt_rep = @m8
      AND t.section_name = N'Срочные'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND ISNULL(t.out_rubm,0) > 0
),
m9 AS (
    SELECT t.cli_id, t.con_id, t.TSEGMENTNAME, t.termdays, t.out_rub, t.out_rubm,
           t.dt_open, t.dt_close_fact, t.dt_close AS dt_close_planned
    FROM alm.[ALM].[vw_balance_rest_all] t
    WHERE t.dt_rep = @m9
      AND t.section_name = N'Срочные'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND ISNULL(t.out_rubm,0) > 0
),

-- 1) Исчезли в АВГУСТЕ: были на 31.07, но их нет на 31.08
aug_disappeared AS (
    SELECT
        p.cli_id, p.con_id, p.TSEGMENTNAME, p.termdays,
        p.out_rub AS out_rub_prev,
        COALESCE(p.dt_close_fact, p.dt_close_planned) AS close_dt
    FROM m7 p
    LEFT JOIN m8 c ON c.con_id = p.con_id
    WHERE c.con_id IS NULL
),
-- 2) Исчезли в СЕНТЯБРЕ: были на 31.08, но их нет на 30.09
sep_disappeared AS (
    SELECT
        p.cli_id, p.con_id, p.TSEGMENTNAME, p.termdays,
        p.out_rub AS out_rub_prev,
        COALESCE(p.dt_close_fact, p.dt_close_planned) AS close_dt
    FROM m8 p
    LEFT JOIN m9 c ON c.con_id = p.con_id
    WHERE c.con_id IS NULL
),

-- 3) Берём только те исчезнувшие, у кого close_dt в целевом окне
exited_window AS (
    SELECT * FROM aug_disappeared
    WHERE close_dt >= @win_start AND close_dt < @win_end_ex
    UNION ALL
    SELECT * FROM sep_disappeared
    WHERE close_dt >= @win_start AND close_dt < @win_end_ex
),

-- 4) Бакетизация
bucketed AS (
    SELECT
        CASE
            WHEN (termdays>=28  AND termdays<=33 ) THEN 31
            WHEN (termdays>=60  AND termdays<=70 ) THEN 61
            WHEN (termdays>=85  AND termdays<=110) THEN 91
            WHEN (termdays>=119 AND termdays<=140) THEN 124
            WHEN (termdays>=175 AND termdays<=200) THEN 181
            WHEN (termdays>=245 AND termdays<=290) THEN 274
            WHEN (termdays>=340 AND termdays<=405) THEN 365
            WHEN (termdays>=540 AND termdays<=621) THEN 550
            WHEN (termdays>=720 AND termdays<=763) THEN 750
            WHEN (termdays>=1090 AND termdays<=1140) THEN 1100
            WHEN (termdays>=1450 AND termdays<=1475) THEN 1460
            WHEN (termdays>=1795 AND termdays<=1830) THEN 1825
            ELSE termdays
        END AS [Бакет срочности],
        NULLIF(LTRIM(RTRIM(TSEGMENTNAME)), N'') AS [Сегмент],
        out_rub_prev AS out_rub
    FROM exited_window
),

-- 5) Агрегация по сегменту и бакету
agg_seg AS (
    SELECT
        [Бакет срочности],
        COALESCE([Сегмент], N'Неопределён') AS [Сегмент],
        SUM(out_rub) AS out_rub_total
    FROM bucketed
    GROUP BY [Бакет срочности], COALESCE([Сегмент], N'Неопределён')
),
-- 6) Итоги «Итого» по каждому бакету
agg_total AS (
    SELECT
        [Бакет срочности],
        N'Итого' AS [Сегмент],
        SUM(out_rub) AS out_rub_total
    FROM bucketed
    GROUP BY [Бакет срочности]
),
unioned AS (
    SELECT * FROM agg_seg
    UNION ALL
    SELECT * FROM agg_total
),
with_share AS (
    SELECT
        [Бакет срочности],
        [Сегмент],
        out_rub_total,
        CAST(out_rub_total * 1.0
             / NULLIF(SUM(out_rub_total) OVER (PARTITION BY [Сегмент]), 0)
             AS decimal(18,6)) AS share_in_segment
    FROM unioned
)
SELECT
    [Бакет срочности],
    [Сегмент],
    out_rub_total                                   AS [Объём выхода],
    CAST(share_in_segment * 100.0 AS decimal(18,2)) AS [Доля бакета внутри сегмента, %]
FROM with_share
ORDER BY
    CASE WHEN [Сегмент] = N'Итого' THEN 2 ELSE 1 END,
    [Сегмент],
    CASE WHEN [Бакет срочности] IS NULL THEN -1 ELSE [Бакет срочности] END;
```

Ключевые моменты:

* **Приоритет анти-джоина:** сначала находим, кто реально исчез с month-end (07→08 и 08→09), затем **фильтруем** по `close_dt` в заданном окне (используем `dt_close_fact`, если есть, иначе `dt_close`).
* **Границы окна:** включительно 19.08.2024 и включительно 15.09.2024 (реализовано как `>= '2024-08-19' AND < '2024-09-16'`).
* **Объём выхода** — баланс на предыдущем срезе (то, что «ушло» в следующем месяце).

Если нужно — добавлю параметризацию под любое произвольное окно (автоматически определю соседние month-end даты для левой/правой части).
