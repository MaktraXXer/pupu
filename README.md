-- === ПАРАМЕТРЫ ===
DECLARE @dt_feb   date = '2025-02-28';
DECLARE @dt_apr   date = '2025-04-01';
DECLARE @m_start  date = '2025-03-01';
DECLARE @m_end_ex date = '2025-04-01';  -- полуинтервал [01.03, 01.04)

WITH
-- 1) Срез на 28.02.2025 — что было (кандидаты на "вышли")
feb AS (
    SELECT
          t.cli_id
        , t.con_id
        , t.TSEGMENTNAME
        , t.termdays
        , t.dt_open
        , t.dt_close_plan         -- если есть в витрине
        , t.out_rub               -- объём на 28.02 (будем считать его "выходом")
        , t.out_rubm
    FROM alm.[ALM].[vw_balance_rest_all] t
    WHERE t.dt_rep = @dt_feb
      AND t.section_name = N'Срочные'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND ISNULL(t.out_rubm, 0) > 0
),
-- 2) Срез на 01.04.2025 — чем "закрыли март" (для anti-join)
apr AS (
    SELECT DISTINCT
          t.con_id
    FROM alm.[ALM].[vw_balance_rest_all] t
    WHERE t.dt_rep = @dt_apr
      AND t.section_name = N'Срочные'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND ISNULL(t.out_rubm, 0) > 0
),
-- 3) Договоры, которые были 28.02, но исчезли к 01.04 → вышли в марте
exited_raw AS (
    SELECT
          f.cli_id
        , f.con_id
        , f.TSEGMENTNAME
        , f.termdays
        , f.dt_open
        , f.dt_close_plan
        , f.out_rub
        , CASE
            WHEN f.dt_close_plan >= @m_start AND f.dt_close_plan < @m_end_ex
                 THEN N'По плану'      -- плановая дата в марте
            ELSE N'Досрочно'           -- план не в марте, а факт исчезновения — в марте
          END AS close_type
    FROM feb f
    LEFT JOIN apr a
      ON a.con_id = f.con_id
    WHERE a.con_id IS NULL  -- не найдено 01.04 => закрыт в марте
),
-- 4) Приводим к бакетам
exited_buckets AS (
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
          close_type,
          out_rub
    FROM exited_raw
),
-- 5) Агрегация по бакетам/сегментам/типу закрытия
agg_seg AS (
    SELECT
          [Бакет срочности],
          COALESCE([Сегмент], N'Неопределён') AS [Сегмент],
          close_type,
          SUM(out_rub) AS out_rub_total,
          COUNT_BIG(*) AS deals_cnt
    FROM exited_buckets
    GROUP BY [Бакет срочности], COALESCE([Сегмент], N'Неопределён'), close_type
),
-- 6) Итоги по сегментам (без разбиения на тип закрытия) — как в прошлой таблице
agg_seg_tot AS (
    SELECT
          [Бакет срочности],
          COALESCE([Сегмент], N'Неопределён') AS [Сегмент],
          N'Итого'::nvarchar(20) AS close_type,
          SUM(out_rub) AS out_rub_total,
          SUM(deals_cnt) AS deals_cnt
    FROM exited_buckets
    GROUP BY [Бакет срочности], COALESCE([Сегмент], N'Неопределён')
),
-- 7) Объединяем (деталь по типу + итоги)
unioned AS (
    SELECT * FROM agg_seg
    UNION ALL
    SELECT * FROM agg_seg_tot
),
-- 8) Доля внутри сегмента (включая «Итого»)
with_share AS (
    SELECT
          [Бакет срочности],
          [Сегмент],
          close_type,
          out_rub_total,
          deals_cnt,
          CAST(out_rub_total * 1.0
               / NULLIF(SUM(out_rub_total) OVER (PARTITION BY [Сегмент]), 0)
               AS decimal(18,6)) AS share_in_segment
    FROM unioned
)
SELECT
      [Бакет срочности],
      [Сегмент],
      close_type               AS [Тип закрытия],       -- По плану / Досрочно / Итого
      out_rub_total            AS [Объём выхода],
      deals_cnt                AS [Кол-во договоров],
      CAST(share_in_segment * 100.0 AS decimal(18,2)) AS [Доля внутри сегмента, %],
      CONCAT(REPLACE(STR(CAST(share_in_segment * 100.0 AS decimal(18,2)), 6, 2), ' ', ''), N'%') AS [Доля (формат)]
FROM with_share
ORDER BY
      CASE WHEN [Сегмент] = N'Итого' THEN 2 ELSE 1 END,
      [Сегмент],
      CASE
          WHEN close_type = N'Итого' THEN 2 ELSE 1 END,
      CASE WHEN [Бакет срочности] IS NULL THEN -1 ELSE [Бакет срочности] END;
