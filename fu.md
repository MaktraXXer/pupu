USE [ALM_TEST];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2025-09-30';
DECLARE @DateTo   date = '2026-08-24';


IF OBJECT_ID('tempdb..#contracts') IS NOT NULL DROP TABLE #contracts;
IF OBJECT_ID('tempdb..#attr_flags') IS NOT NULL DROP TABLE #attr_flags;
IF OBJECT_ID('tempdb..#contract_category') IS NOT NULL DROP TABLE #contract_category;
IF OBJECT_ID('tempdb..#daily_delta') IS NOT NULL DROP TABLE #daily_delta;


/* ============================================================
   1. ДОГОВОРЫ

   DepositInterestsRateSnap — реестр интересующих нас вкладов.

   По каждому CON_ID оставляем одну, наиболее свежую запись
   на конец анализируемого периода.

   Здесь же сохраняем:
   - PROD_NAME для ФУ;
   - isfloat для плавающего вклада.
   ============================================================ */

WITH contract_ranked AS
(
    SELECT
          CAST(d.CON_ID AS bigint) AS con_id
        , d.PROD_NAME

        , CASE
              WHEN ISNULL(TRY_CAST(d.isfloat AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_float_flag

        , ROW_NUMBER() OVER
          (
              PARTITION BY d.CON_ID
              ORDER BY d.DT_REP DESC
          ) AS rn

    FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap] d WITH (NOLOCK)

    WHERE
        d.DT_REP <= @DateTo

        /* Отсекаем сделки, которые точно ещё не существовали */
        AND d.DT_OPEN <= @DateTo

        /* Отсекаем сделки, которые точно закрылись
           до начала анализируемого диапазона */
        AND
        (
            d.DT_CLOSE IS NULL
            OR d.DT_CLOSE >= @DateFrom
        )
)

SELECT
      con_id
    , PROD_NAME
    , is_float_flag
INTO #contracts
FROM contract_ranked
WHERE rn = 1;


CREATE UNIQUE CLUSTERED INDEX IX_contracts
ON #contracts (con_id);


/* ============================================================
   2. ДОПОЛНИТЕЛЬНЫЕ ПРИЗНАКИ ДОГОВОРА

   Только для договоров из #contracts.

   По каждому CON_ID:
   последняя запись по DT_UPDATE, затем loaddate.
   ============================================================ */

WITH attr_ranked AS
(
    SELECT
          CAST(a.CON_ID AS bigint) AS con_id

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

    FROM [ALM].[ehd].[attr_DepoFLConditions] a WITH (NOLOCK)

    INNER JOIN #contracts c
        ON c.con_id = a.CON_ID
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
   3. КЛАССИФИКАЦИЯ ДОГОВОРА

   Здесь каждый CON_ID получает ОДНУ категорию НАВСЕГДА
   для данного расчёта.

   Приоритет:

   1. FLOAT
   2. ФУ
   3. Нов
   4. Пр2 / Пр3
   5. НДП / НДМ
   6. Пк2 / От1
   7. Пк3 / Пк6
   8. МПЛ
   9. Пнс
   10. Остальные
   ============================================================ */

SELECT
      c.con_id

    , CASE
          /* 1. Плавающий */
          WHEN c.is_float_flag = 1
              THEN 'float'

          /* 2. Финуслуги */
          WHEN c.PROD_NAME IN
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
              THEN 'fu'

          /* 3. Нов */
          WHEN ISNULL(a.is_nov_flag, 0) = 1
              THEN 'nov'

          /* 4. Пр2 / Пр3 */
          WHEN ISNULL(a.is_pr2_pr3_flag, 0) = 1
              THEN 'pr2_pr3'

          /* 5. НДП / НДМ */
          WHEN ISNULL(a.is_ndp_ndm_flag, 0) = 1
              THEN 'ndp_ndm'

          /* 6. Пк2 / От1 */
          WHEN ISNULL(a.is_pk2_ot1_flag, 0) = 1
              THEN 'pk2_ot1'

          /* 7. Пк3 / Пк6 */
          WHEN ISNULL(a.is_pk3_pk6_flag, 0) = 1
              THEN 'pk3_pk6'

          /* 8. МПЛ */
          WHEN ISNULL(a.is_mpl_flag, 0) = 1
              THEN 'mpl'

          /* 9. Пнс */
          WHEN ISNULL(a.is_pns_flag, 0) = 1
              THEN 'pns'

          ELSE 'other'

      END AS deposit_category

INTO #contract_category

FROM #contracts c

LEFT JOIN #attr_flags a
    ON a.con_id = c.con_id;


CREATE UNIQUE CLUSTERED INDEX IX_contract_category
ON #contract_category (con_id);


/* ============================================================
   4. ИСТОРИЯ САЛЬДО

   КЛЮЧЕВАЯ ОПТИМИЗАЦИЯ.

   НЕ делаем:

       saldo
       JOIN calendar
       ON date BETWEEN DT_FROM AND DT_TO

   Потому что это может породить огромный объём строк.

   Вместо этого каждая строка saldo превращается:

       DT_FROM       +OUT_RUB
       DT_TO + 1     -OUT_RUB

   Если интервал начался раньше @DateFrom,
   +OUT_RUB ставим на @DateFrom.

   После этого накопительная сумма = баланс на дату.
   ============================================================ */

CREATE TABLE #daily_delta
(
      dt_rep date NOT NULL PRIMARY KEY

    , delta_float decimal(38,6) NOT NULL
    , delta_fu decimal(38,6) NOT NULL
    , delta_nov decimal(38,6) NOT NULL
    , delta_pr2_pr3 decimal(38,6) NOT NULL
    , delta_ndp_ndm decimal(38,6) NOT NULL
    , delta_pk2_ot1 decimal(38,6) NOT NULL
    , delta_pk3_pk6 decimal(38,6) NOT NULL
    , delta_mpl decimal(38,6) NOT NULL
    , delta_pns decimal(38,6) NOT NULL
    , delta_other decimal(38,6) NOT NULL
);


WITH saldo_events AS
(
    /* ========================================================
       НАЧАЛО ИНТЕРВАЛА
       ======================================================== */

    SELECT
          CASE
              WHEN s.DT_FROM < @DateFrom
                  THEN @DateFrom
              ELSE CAST(s.DT_FROM AS date)
          END AS event_date

        , cc.deposit_category

        , CAST(s.OUT_RUB AS decimal(38,6)) AS delta

    FROM [LIQUIDITY].[liq].[DepositContract_Saldo] s WITH (NOLOCK)

    INNER JOIN #contract_category cc
        ON cc.con_id = s.CON_ID

    WHERE
        s.DT_FROM <= @DateTo

        AND
        (
            s.DT_TO IS NULL
            OR s.DT_TO >= @DateFrom
        )

        AND s.OUT_RUB IS NOT NULL


    UNION ALL


    /* ========================================================
       КОНЕЦ ИНТЕРВАЛА

       Если DT_TO = 30.10,
       остаток действует ВКЛЮЧИТЕЛЬНО 30.10.

       Поэтому вычитаем его 31.10.
       ======================================================== */

    SELECT
          DATEADD(day, 1, CAST(s.DT_TO AS date))
              AS event_date

        , cc.deposit_category

        , -CAST(s.OUT_RUB AS decimal(38,6))
              AS delta

    FROM [LIQUIDITY].[liq].[DepositContract_Saldo] s WITH (NOLOCK)

    INNER JOIN #contract_category cc
        ON cc.con_id = s.CON_ID

    WHERE
        s.DT_FROM <= @DateTo

        AND s.DT_TO IS NOT NULL

        AND s.DT_TO >= @DateFrom

        /* События после @DateTo уже не нужны */
        AND s.DT_TO < @DateTo

        AND s.OUT_RUB IS NOT NULL
),

events_agg AS
(
    SELECT
          event_date

        , SUM(
              CASE WHEN deposit_category = 'float'
                   THEN delta ELSE 0 END
          ) AS delta_float

        , SUM(
              CASE WHEN deposit_category = 'fu'
                   THEN delta ELSE 0 END
          ) AS delta_fu

        , SUM(
              CASE WHEN deposit_category = 'nov'
                   THEN delta ELSE 0 END
          ) AS delta_nov

        , SUM(
              CASE WHEN deposit_category = 'pr2_pr3'
                   THEN delta ELSE 0 END
          ) AS delta_pr2_pr3

        , SUM(
              CASE WHEN deposit_category = 'ndp_ndm'
                   THEN delta ELSE 0 END
          ) AS delta_ndp_ndm

        , SUM(
              CASE WHEN deposit_category = 'pk2_ot1'
                   THEN delta ELSE 0 END
          ) AS delta_pk2_ot1

        , SUM(
              CASE WHEN deposit_category = 'pk3_pk6'
                   THEN delta ELSE 0 END
          ) AS delta_pk3_pk6

        , SUM(
              CASE WHEN deposit_category = 'mpl'
                   THEN delta ELSE 0 END
          ) AS delta_mpl

        , SUM(
              CASE WHEN deposit_category = 'pns'
                   THEN delta ELSE 0 END
          ) AS delta_pns

        , SUM(
              CASE WHEN deposit_category = 'other'
                   THEN delta ELSE 0 END
          ) AS delta_other

    FROM saldo_events

    WHERE
        event_date >= @DateFrom
        AND event_date <= @DateTo

    GROUP BY
        event_date
)

INSERT INTO #daily_delta
(
      dt_rep
    , delta_float
    , delta_fu
    , delta_nov
    , delta_pr2_pr3
    , delta_ndp_ndm
    , delta_pk2_ot1
    , delta_pk3_pk6
    , delta_mpl
    , delta_pns
    , delta_other
)

SELECT
      event_date
    , delta_float
    , delta_fu
    , delta_nov
    , delta_pr2_pr3
    , delta_ndp_ndm
    , delta_pk2_ot1
    , delta_pk3_pk6
    , delta_mpl
    , delta_pns
    , delta_other
FROM events_agg;


/* ============================================================
   5. КАЛЕНДАРЬ

   Он маленький — максимум несколько сотен строк.
   ============================================================ */

;WITH calendar AS
(
    SELECT
        @DateFrom AS dt_rep

    UNION ALL

    SELECT
        DATEADD(day, 1, dt_rep)

    FROM calendar

    WHERE dt_rep < @DateTo
),

daily AS
(
    SELECT
          c.dt_rep

        , ISNULL(d.delta_float, 0)
            AS delta_float

        , ISNULL(d.delta_fu, 0)
            AS delta_fu

        , ISNULL(d.delta_nov, 0)
            AS delta_nov

        , ISNULL(d.delta_pr2_pr3, 0)
            AS delta_pr2_pr3

        , ISNULL(d.delta_ndp_ndm, 0)
            AS delta_ndp_ndm

        , ISNULL(d.delta_pk2_ot1, 0)
            AS delta_pk2_ot1

        , ISNULL(d.delta_pk3_pk6, 0)
            AS delta_pk3_pk6

        , ISNULL(d.delta_mpl, 0)
            AS delta_mpl

        , ISNULL(d.delta_pns, 0)
            AS delta_pns

        , ISNULL(d.delta_other, 0)
            AS delta_other

    FROM calendar c

    LEFT JOIN #daily_delta d
        ON d.dt_rep = c.dt_rep
),

balances AS
(
    SELECT
          dt_rep

        /* 1. FLOAT */
        , SUM(delta_float) OVER
          (
              ORDER BY dt_rep
              ROWS UNBOUNDED PRECEDING
          ) AS balance_float

        /* 2. ФУ */
        , SUM(delta_fu) OVER
          (
              ORDER BY dt_rep
              ROWS UNBOUNDED PRECEDING
          ) AS balance_fu

        /* 3. НОВ */
        , SUM(delta_nov) OVER
          (
              ORDER BY dt_rep
              ROWS UNBOUNDED PRECEDING
          ) AS balance_nov

        /* 4. Пр2 / Пр3 */
        , SUM(delta_pr2_pr3) OVER
          (
              ORDER BY dt_rep
              ROWS UNBOUNDED PRECEDING
          ) AS balance_pr2_pr3

        /* 5. НДП / НДМ */
        , SUM(delta_ndp_ndm) OVER
          (
              ORDER BY dt_rep
              ROWS UNBOUNDED PRECEDING
          ) AS balance_ndp_ndm

        /* 6. Пк2 / От1 */
        , SUM(delta_pk2_ot1) OVER
          (
              ORDER BY dt_rep
              ROWS UNBOUNDED PRECEDING
          ) AS balance_pk2_ot1

        /* 7. Пк3 / Пк6 */
        , SUM(delta_pk3_pk6) OVER
          (
              ORDER BY dt_rep
              ROWS UNBOUNDED PRECEDING
          ) AS balance_pk3_pk6

        /* 8. МПЛ */
        , SUM(delta_mpl) OVER
          (
              ORDER BY dt_rep
              ROWS UNBOUNDED PRECEDING
          ) AS balance_mpl

        /* 9. Пнс */
        , SUM(delta_pns) OVER
          (
              ORDER BY dt_rep
              ROWS UNBOUNDED PRECEDING
          ) AS balance_pns

        /* Остальные */
        , SUM(delta_other) OVER
          (
              ORDER BY dt_rep
              ROWS UNBOUNDED PRECEDING
          ) AS balance_other

    FROM daily
)


/* ============================================================
   6. ИТОГ

   Одна строка = одна календарная дата.
   ============================================================ */

SELECT
      dt_rep

    , balance_float
    , balance_fu
    , balance_nov
    , balance_pr2_pr3
    , balance_ndp_ndm
    , balance_pk2_ot1
    , balance_pk3_pk6
    , balance_mpl
    , balance_pns
    , balance_other


    /* Весь баланс */
    , balance_float
      + balance_fu
      + balance_nov
      + balance_pr2_pr3
      + balance_ndp_ndm
      + balance_pk2_ot1
      + balance_pk3_pk6
      + balance_mpl
      + balance_pns
      + balance_other
        AS balance_total

FROM balances

ORDER BY
    dt_rep

OPTION (MAXRECURSION 0);
