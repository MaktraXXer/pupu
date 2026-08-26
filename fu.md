USE [ALM];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2025-09-30';
DECLARE @DateTo   date = '2026-08-24';


IF OBJECT_ID('tempdb..#relevant_con_ids') IS NOT NULL
    DROP TABLE #relevant_con_ids;

IF OBJECT_ID('tempdb..#attr_flags') IS NOT NULL
    DROP TABLE #attr_flags;


/* ============================================================
   1. Собираем только con_id вкладов,
      которые вообще встречаются в нужном диапазоне.

   Сам баланс целиком в tempdb НЕ складываем.
   ============================================================ */
SELECT DISTINCT
      CAST(t.con_id AS bigint) AS con_id
INTO #relevant_con_ids
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE
    t.dt_rep >= @DateFrom
    AND t.dt_rep <= @DateTo

    AND t.section_name = N'Срочные'
    AND t.block_name = N'Привлечение ФЛ'
    AND t.acc_role   = N'LIAB'
    AND t.od_flag    = 1
    AND t.cur        = '810'

    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
    AND t.con_id IS NOT NULL;


CREATE UNIQUE CLUSTERED INDEX IX_relevant_con_ids
ON #relevant_con_ids (con_id);


/* ============================================================
   2. Последние признаки договора

   По каждому con_id:
      DT_UPDATE DESC
      loaddate DESC

   ФУ и floatrate здесь не нужны:
   они определяются непосредственно из баланса.
   ============================================================ */
WITH attr_ranked AS
(
    SELECT
          TRY_CAST(a.CON_ID AS bigint) AS con_id

        /* Нов */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Нов] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_nov_flag

        /* Пр2 / Пр3 */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Пр2] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Пр3] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_pr2_pr3_flag

        /* НДП / НДМ */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[НДП] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[НДМ] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_ndp_ndm_flag

        /* Пк2 / От1 */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Пк2] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[От1] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_pk2_ot1_flag

        /* Пк3 / Пк6 */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Пк3] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Пк6] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_pk3_pk6_flag

        /* МПЛ */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Мпл] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_mpl_flag

        /* Пнс */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Пнс] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_pns_flag

        , ROW_NUMBER() OVER
          (
              PARTITION BY a.CON_ID
              ORDER BY
                    a.DT_UPDATE DESC
                  , a.loaddate DESC
          ) AS rn

    FROM alm.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    INNER JOIN #relevant_con_ids r
        ON r.con_id = TRY_CAST(a.CON_ID AS bigint)
)

SELECT
      con_id
    , is_nov_flag
    , is_pr2_pr3_flag
    , is_ndp_ndm_flag
    , is_pk2_ot1_flag
    , is_pk3_pk6_flag
    , is_mpl_flag
    , is_pns_flag
INTO #attr_flags
FROM attr_ranked
WHERE rn = 1;


CREATE UNIQUE CLUSTERED INDEX IX_attr_flags
ON #attr_flags (con_id);


/* ============================================================
   3. Классификация всего баланса вкладов

   ПРИОРИТЕТ:

   1. FLOAT
   2. ФУ
   3. Нов
   4. Пр2 / Пр3
   5. НДП / НДМ
   6. Пк2 / От1
   7. Пк3 / Пк6
   8. МПЛ
   9. Пнс
   10. Остальные — технически считаем, но в основной
       выдаче ниже не показываем.
   ============================================================ */
WITH balance_classified AS
(
    SELECT
          CAST(t.dt_rep AS date) AS dt_rep
        , CAST(t.out_rub AS decimal(38,6)) AS out_rub

        , CASE

              /* 1. Плавающий вклад — ВСЕГДА первый приоритет */
              WHEN ISNULL(TRY_CAST(t.is_floatrate AS int), 0) = 1
                  THEN N'float'

              /* 2. Финуслуги */
              WHEN t.PROD_NAME_res IN
              (
                    N'Надёжный прайм'
                  , N'Надёжный VIP'
                  , N'Надёжный премиум'
                  , N'Надёжный промо'
                  , N'Надёжный старт'
                  , N'Надёжный Т2'
                  , N'Надёжный Мегафон'
                  , N'Надёжный процент'
                  , N'Надёжныйпроцент'
                  , N'Могучий'
                  , N'Надёжный'
              )
                  THEN N'fu'

              /* 3. Нов */
              WHEN ISNULL(a.is_nov_flag, 0) = 1
                  THEN N'nov'

              /* 4. Пр2 / Пр3 */
              WHEN ISNULL(a.is_pr2_pr3_flag, 0) = 1
                  THEN N'pr2_pr3'

              /* 5. НДП / НДМ */
              WHEN ISNULL(a.is_ndp_ndm_flag, 0) = 1
                  THEN N'ndp_ndm'

              /* 6. Пк2 / От1 */
              WHEN ISNULL(a.is_pk2_ot1_flag, 0) = 1
                  THEN N'pk2_ot1'

              /* 7. Пк3 / Пк6 */
              WHEN ISNULL(a.is_pk3_pk6_flag, 0) = 1
                  THEN N'pk3_pk6'

              /* 8. МПЛ */
              WHEN ISNULL(a.is_mpl_flag, 0) = 1
                  THEN N'mpl'

              /* 9. Пнс */
              WHEN ISNULL(a.is_pns_flag, 0) = 1
                  THEN N'pns'

              ELSE N'other'

          END AS deposit_category

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    LEFT JOIN #attr_flags a
        ON a.con_id = CAST(t.con_id AS bigint)

    WHERE
        t.dt_rep >= @DateFrom
        AND t.dt_rep <= @DateTo

        AND t.section_name = N'Срочные'
        AND t.block_name = N'Привлечение ФЛ'
        AND t.acc_role   = N'LIAB'
        AND t.od_flag    = 1
        AND t.cur        = '810'

        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
)


/* ============================================================
   4. Результат

   Одна строка = одна дата баланса.
   9 нужных категорий = 9 отдельных объёмов.
   ============================================================ */
SELECT
      dt_rep

    , SUM(
          CASE WHEN deposit_category = N'float'
               THEN out_rub ELSE 0 END
      ) AS balance_float

    , SUM(
          CASE WHEN deposit_category = N'fu'
               THEN out_rub ELSE 0 END
      ) AS balance_fu

    , SUM(
          CASE WHEN deposit_category = N'nov'
               THEN out_rub ELSE 0 END
      ) AS balance_nov

    , SUM(
          CASE WHEN deposit_category = N'pr2_pr3'
               THEN out_rub ELSE 0 END
      ) AS balance_pr2_pr3

    , SUM(
          CASE WHEN deposit_category = N'ndp_ndm'
               THEN out_rub ELSE 0 END
      ) AS balance_ndp_ndm

    , SUM(
          CASE WHEN deposit_category = N'pk2_ot1'
               THEN out_rub ELSE 0 END
      ) AS balance_pk2_ot1

    , SUM(
          CASE WHEN deposit_category = N'pk3_pk6'
               THEN out_rub ELSE 0 END
      ) AS balance_pk3_pk6

    , SUM(
          CASE WHEN deposit_category = N'mpl'
               THEN out_rub ELSE 0 END
      ) AS balance_mpl

    , SUM(
          CASE WHEN deposit_category = N'pns'
               THEN out_rub ELSE 0 END
      ) AS balance_pns

FROM balance_classified

GROUP BY
    dt_rep

ORDER BY
    dt_rep

OPTION (RECOMPILE);
