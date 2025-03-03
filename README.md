-----------------------------------------------------------------------
-- 1) Базовый WITH final: формируем цепочку до p5, определяем ProlongCount.
--    Отбираем только RUR (dc.CUR = 'RUR').
-----------------------------------------------------------------------
WITH final AS (
    SELECT
         dc.CON_ID         AS CurrentConId,
         dc.CUR,
         dc.BALANCE_RUB,
         dc.DT_OPEN,
         dc.DT_CLOSE,
         -- 0..5 уровень пролонгации
         CASE
           WHEN p1.CON_ID IS NULL OR old1.CON_ID IS NULL THEN 0
           WHEN p2.CON_ID IS NULL OR old2.CON_ID IS NULL THEN 1
           WHEN p3.CON_ID IS NULL OR old3.CON_ID IS NULL THEN 2
           WHEN p4.CON_ID IS NULL OR old4.CON_ID IS NULL THEN 3
           WHEN p5.CON_ID IS NULL OR old5.CON_ID IS NULL THEN 4
           ELSE 5
         END AS ProlongCount,
         
         -- Ссылки на предыдущие вклады (old1..old5), чтобы проверять их длительность
         old1.CON_ID AS Old1_ConId,
         old1.DT_OPEN AS Old1_Open,
         old1.DT_CLOSE AS Old1_Close,
         
         old2.CON_ID AS Old2_ConId,
         old2.DT_OPEN AS Old2_Open,
         old2.DT_CLOSE AS Old2_Close,
         
         old3.CON_ID AS Old3_ConId,
         old3.DT_OPEN AS Old3_Open,
         old3.DT_CLOSE AS Old3_Close,
         
         old4.CON_ID AS Old4_ConId,
         old4.DT_OPEN AS Old4_Open,
         old4.DT_CLOSE AS Old4_Close,
         
         old5.CON_ID AS Old5_ConId,
         old5.DT_OPEN AS Old5_Open,
         old5.DT_CLOSE AS Old5_Close

    FROM [LIQUIDITY].[liq].[DepositContract_all] dc
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p1
           ON p1.CON_ID = dc.CON_ID
          AND p1.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old1
           ON old1.CON_ID = p1.CON_ID_REL
    
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p2
           ON p2.CON_ID = old1.CON_ID
          AND p2.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old2
           ON old2.CON_ID = p2.CON_ID_REL
    
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p3
           ON p3.CON_ID = old2.CON_ID
          AND p3.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old3
           ON old3.CON_ID = p3.CON_ID_REL
    
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p4
           ON p4.CON_ID = old3.CON_ID
          AND p4.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old4
           ON old4.CON_ID = p4.CON_ID_REL
    
    LEFT JOIN [ALM].[ehd].[conrel_prolongations] p5
           ON p5.CON_ID = old4.CON_ID
          AND p5.CON_REL_TYPE = 'PREVIOUS'
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_all] old5
           ON old5.CON_ID = p5.CON_ID_REL
    
    -- Фильтрация только рублёвых вкладов
    WHERE dc.CUR = 'RUR'
)
-----------------------------------------------------------------------
-- 2) Список всех сценариев - Prolongation=1..5
--    Проверка: old1..oldN <10 days?
--    Для каждого сценария выводим dealsCount + sampleDeal.
--    Можно упорядочить по scenario.
-----------------------------------------------------------------------
SELECT
    'Prolongation=1 but old1<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 1
  AND Old1_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old1_Open, Old1_Close) < 10

UNION ALL

-----------------------------------------------------------------------
-- ProlongCount=2, проверяем old1 и old2
-----------------------------------------------------------------------
SELECT
    'Prolongation=2 but old1<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 2
  AND Old1_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old1_Open, Old1_Close) < 10

UNION ALL

SELECT
    'Prolongation=2 but old2<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 2
  AND Old2_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old2_Open, Old2_Close) < 10

-----------------------------------------------------------------------
-- ProlongCount=3, проверяем old1..old3
-----------------------------------------------------------------------
UNION ALL
SELECT
    'Prolongation=3 but old1<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 3
  AND Old1_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old1_Open, Old1_Close) < 10

UNION ALL
SELECT
    'Prolongation=3 but old2<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 3
  AND Old2_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old2_Open, Old2_Close) < 10

UNION ALL
SELECT
    'Prolongation=3 but old3<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 3
  AND Old3_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old3_Open, Old3_Close) < 10

-----------------------------------------------------------------------
-- ProlongCount=4, проверяем old1..old4
-----------------------------------------------------------------------
UNION ALL
SELECT
    'Prolongation=4 but old1<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 4
  AND Old1_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old1_Open, Old1_Close) < 10

UNION ALL
SELECT
    'Prolongation=4 but old2<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 4
  AND Old2_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old2_Open, Old2_Close) < 10

UNION ALL
SELECT
    'Prolongation=4 but old3<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 4
  AND Old3_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old3_Open, Old3_Close) < 10

UNION ALL
SELECT
    'Prolongation=4 but old4<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 4
  AND Old4_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old4_Open, Old4_Close) < 10

-----------------------------------------------------------------------
-- ProlongCount=5, проверяем old1..old5
-----------------------------------------------------------------------
UNION ALL
SELECT
    'Prolongation=5 but old1<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 5
  AND Old1_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old1_Open, Old1_Close) < 10

UNION ALL
SELECT
    'Prolongation=5 but old2<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 5
  AND Old2_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old2_Open, Old2_Close) < 10

UNION ALL
SELECT
    'Prolongation=5 but old3<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 5
  AND Old3_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old3_Open, Old3_Close) < 10

UNION ALL
SELECT
    'Prolongation=5 but old4<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 5
  AND Old4_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old4_Open, Old4_Close) < 10

UNION ALL
SELECT
    'Prolongation=5 but old5<10' AS scenario,
    COUNT(*) AS dealsCount,
    MIN(final.CurrentConId) AS sampleDeal
FROM final
WHERE ProlongCount = 5
  AND Old5_ConId IS NOT NULL
  AND DATEDIFF(DAY, Old5_Open, Old5_Close) < 10

-----------------------------------------------------------------------
ORDER BY scenario;
