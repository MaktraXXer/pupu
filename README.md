-- === Параметры ===
DECLARE @win_start  date = '2024-08-19';   -- включительно
DECLARE @win_end_ex date = '2024-09-16';   -- полуинтервал (до 16.09)
DECLARE @dt_rep     date = '2024-09-30';   -- интересует объём на 30 сентября

WITH base_0930 AS (
    SELECT
          t.cli_id
        , t.con_id
        , t.TSEGMENTNAME
        , t.termdays
        , t.dt_open
        , t.out_rub
    FROM alm.[ALM].[vw_balance_rest_all] t
    WHERE t.dt_rep       = @dt_rep
      AND t.section_name = N'Срочные'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND ISNULL(t.is_floatrate,0) = 0
      AND ISNULL(t.out_rub,0) > 0
),
filtered AS (
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
        out_rub
    FROM base_0930
    WHERE dt_open >= @win_start
      AND dt_open <  @win_end_ex
),
agg_seg AS (
    SELECT
          [Бакет срочности],
          COALESCE([Сегмент], N'Неопределён') AS [Сегмент],
          SUM(out_rub) AS out_rub_total
    FROM filtered
    GROUP BY [Бакет срочности], COALESCE([Сегмент], N'Неопределён')
),
agg_total AS (
    SELECT
          [Бакет срочности],
          N'Итого' AS [Сегмент],
          SUM(out_rub) AS out_rub_total
    FROM filtered
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
      out_rub_total                                   AS [Объём на 30.09],
      CAST(share_in_segment * 100.0 AS decimal(18,2)) AS [Доля бакета внутри сегмента, %]
FROM with_share
ORDER BY
      CASE WHEN [Сегмент] = N'Итого' THEN 2 ELSE 1 END,
      [Сегмент],
      CASE WHEN [Бакет срочности] IS NULL THEN -1 ELSE [Бакет срочности] END;
