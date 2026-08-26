USE [ALM_TEST];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2025-09-30';
DECLARE @DateTo   date = '2026-08-24';


IF OBJECT_ID('tempdb..#contracts') IS NOT NULL
    DROP TABLE #contracts;

IF OBJECT_ID('tempdb..#attr_flags') IS NOT NULL
    DROP TABLE #attr_flags;

IF OBJECT_ID('tempdb..#contract_category') IS NOT NULL
    DROP TABLE #contract_category;

IF OBJECT_ID('tempdb..#daily_delta') IS NOT NULL
    DROP TABLE #daily_delta;


/* ============================================================
   1. РЕЕСТР ДОГОВОРОВ

   DepositInterestsRateSnap используем как реестр вкладов.

   DT_REP здесь НЕ является периодом действия остатка.

   Для каждого CON_ID берём последнюю известную запись,
   чтобы получить актуальные атрибуты самого договора:
   - DT_OPEN
   - DT_CLOSE
   - PROD_NAME
   - isfloat

   После этого оставляем только договоры, жизнь которых
   пересекается с @DateFrom ... @DateTo.
   ============================================================ */
WITH contract_ranked AS
(
    SELECT
          CAST(d.CON_ID AS bigint) AS con_id
        , CAST(d.DT_OPEN AS date) AS dt_open
        , CAST(d.DT_CLOSE AS date) AS dt_close
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

    /* Нет смысла тащить договоры,
       которые открылись уже после нашего периода */
    WHERE
        d.DT_OPEN <= @DateTo
),

contracts_filtered AS
(
    SELECT
          con_id
        , dt_open
        , dt_close
        , PROD_NAME
        , is_float_flag

    FROM contract_ranked

    WHERE
        rn = 1

        /* Жизнь договора пересекает анализируемый период */
        AND dt_open <= @DateTo

        AND
        (
            dt_close IS NULL
            OR dt_close >= @DateFrom
        )
)

SELECT
      con_id
    , dt_open
    , dt_close
    , PROD_NAME
    , is_float_flag

INTO #contracts

FROM contracts_filtered;


CREATE UNIQUE CLUSTERED INDEX IX_contracts_con_id
ON #contracts (con_id);


/* ============================================================
   2. ДОПОЛНИТЕЛЬНЫЕ ПРИЗНАКИ ДОГОВОРОВ

   Берём attr_DepoFLConditions только для интересующих CON_ID.

   На каждый CON_ID берём последнюю запись:
   1. DT_UPDATE DESC
   2. loaddate DESC
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


        /* Пр2 + Пр3 */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Пр2] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[Пр3] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_pr2_pr3_flag


        /* НДП + НДМ */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[НДП] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[НДМ] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_ndp_ndm_flag


        /* Пк2 + От1 */
        , CASE
              WHEN ISNULL(TRY_CAST(a.[Пк2] AS int), 0) = 1
                OR ISNULL(TRY_CAST(a.[От1] AS int), 0) = 1
                  THEN 1
              ELSE 0
          END AS is_pk2_ot1_flag


        /* Пк3 + Пк6 */
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
        ON a.CON_ID = c.con_id
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


CREATE UNIQUE CLUSTERED INDEX IX_attr_flags_con_id
ON #attr_flags (con_id);


/* ============================================================
   3. КАЖДОМУ ДОГОВОРУ НАЗНАЧАЕМ ОДНУ КАТЕГОРИЮ

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
    , c.dt_open
    , c.dt_close

    , CASE

          /* 1. Плавающий вклад */
          WHEN c.is_float_flag = 1
              THEN N'float'


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


          /* 10. Остальные */
          ELSE N'other'

      END AS deposit_category

INTO #contract_category

FROM #contracts c

LEFT JOIN #attr_flags a
    ON a.con_id = c.con_id;


CREATE UNIQUE CLUSTERED INDEX IX_contract_category_con_id
ON #contract_category (con_id);


/* ============================================================
   4. СОБЫТИЯ ИЗ ИСТОРИИ САЛЬДО

   DepositContract_Saldo задаёт фактический остаток:

       DT_FROM ... DT_TO = OUT_RUB

   Но дополнительно ограничиваем этот интервал:
   - жизнью договора DT_OPEN ... DT_CLOSE
   - нашим анализируемым периодом @DateFrom ... @DateTo

   Вместо ежедневного размножения строки создаём:

       effective_from     +OUT_RUB
       effective_to + 1   -OUT_RUB

   То есть каждая запись сальдо превращается максимум
   в две строки.
   ============================================================ */

CREATE TABLE #daily_delta
(
      dt_rep date NOT NULL
        PRIMARY KEY CLUSTERED

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


/* ============================================================
   Вычисляем пересечение:

   effective_from =
       MAX(DT_FROM, DT_OPEN, @DateFrom)

   effective_to =
       MIN(DT_TO, DT_CLOSE, @DateTo)
   ============================================================ */
;WITH saldo_clipped AS
(
    SELECT
          cc.con_id
        , cc.deposit_category
        , CAST(s.OUT_RUB AS decimal(38,6)) AS out_rub

        , bounds_from.effective_from
        , bounds_to.effective_to

    FROM [LIQUIDITY].[liq].[DepositContract_Saldo] s WITH (NOLOCK)

    INNER JOIN #contract_category cc
        ON cc.con_id = s.CON_ID


    /* ========================================================
       Самый ранний возможный effective_from
       ======================================================== */
    CROSS APPLY
    (
        SELECT MAX(v.dt) AS effective_from

        FROM
        (
            VALUES
                  (CAST(s.DT_FROM AS date))
                , (cc.dt_open)
                , (@DateFrom)
        ) v(dt)

    ) bounds_from


    /* ========================================================
       Самый ранний effective_to
       ======================================================== */
    CROSS APPLY
    (
        SELECT MIN(v.dt) AS effective_to

        FROM
        (
            VALUES
                  (
                      ISNULL(
                          CAST(s.DT_TO AS date),
                          @DateTo
                      )
                  )
                , (
                      ISNULL(
                          cc.dt_close,
                          @DateTo
                      )
                  )
                , (@DateTo)
        ) v(dt)

    ) bounds_to


    WHERE
        /* Сальдо вообще пересекает нужный период */
        s.DT_FROM <= @DateTo

        AND
        (
            s.DT_TO IS NULL
            OR s.DT_TO >= @DateFrom
        )

        AND s.OUT_RUB IS NOT NULL


        /* После всех ограничений интервал действительно существует */
        AND bounds_from.effective_from
            <= bounds_to.effective_to
),


/* ============================================================
   Каждая строка saldo -> максимум два события.

   Это делается через CROSS APPLY,
   поэтому DepositContract_Saldo физически не нужно
   читать второй раз.
   ============================================================ */
saldo_events AS
(
    SELECT
          e.event_date
        , s.deposit_category
        , e.delta

    FROM saldo_clipped s

    CROSS APPLY
    (
        VALUES

        /* Начало действия остатка */
        (
              s.effective_from
            , s.out_rub
        )

        /* На следующий день после конца действия
           остаток снимается */
        ,
        (
              CASE
                  WHEN s.effective_to < @DateTo
                      THEN DATEADD(day, 1, s.effective_to)
                  ELSE NULL
              END

            , -s.out_rub
        )

    ) e(event_date, delta)

    WHERE
        e.event_date IS NOT NULL
),


/* ============================================================
   Схлопываем все события одной даты сразу до 10 категорий.

   После этого объём данных становится очень маленьким:
   максимум одна строка на календарную дату.
   ============================================================ */
events_agg AS
(
    SELECT
          event_date


        , SUM(
              CASE
                  WHEN deposit_category = N'float'
                      THEN delta
                  ELSE 0
              END
          ) AS delta_float


        , SUM(
              CASE
                  WHEN deposit_category = N'fu'
                      THEN delta
                  ELSE 0
              END
          ) AS delta_fu


        , SUM(
              CASE
                  WHEN deposit_category = N'nov'
                      THEN delta
                  ELSE 0
              END
          ) AS delta_nov


        , SUM(
              CASE
                  WHEN deposit_category = N'pr2_pr3'
                      THEN delta
                  ELSE 0
              END
          ) AS delta_pr2_pr3


        , SUM(
              CASE
                  WHEN deposit_category = N'ndp_ndm'
                      THEN delta
                  ELSE 0
              END
          ) AS delta_ndp_ndm


        , SUM(
              CASE
                  WHEN deposit_category = N'pk2_ot1'
                      THEN delta
                  ELSE 0
              END
          ) AS delta_pk2_ot1


        , SUM(
              CASE
                  WHEN deposit_category = N'pk3_pk6'
                      THEN delta
                  ELSE 0
              END
          ) AS delta_pk3_pk6


        , SUM(
              CASE
                  WHEN deposit_category = N'mpl'
                      THEN delta
                  ELSE 0
              END
          ) AS delta_mpl


        , SUM(
              CASE
                  WHEN deposit_category = N'pns'
                      THEN delta
                  ELSE 0
              END
          ) AS delta_pns


        , SUM(
              CASE
                  WHEN deposit_category = N'other'
                      THEN delta
                  ELSE 0
              END
          ) AS delta_other


    FROM saldo_events

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
   5. КАЛЕНДАРЬ + НАКОПИТЕЛЬНЫЙ ИТОГ

   Теперь тяжёлые исходные таблицы больше не нужны.

   В #daily_delta максимум одна строка на дату.

   По событиям восстанавливаем фактический остаток
   каждой категории на каждый день.
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

        , ISNULL(d.delta_float, 0) AS delta_float
        , ISNULL(d.delta_fu, 0) AS delta_fu
        , ISNULL(d.delta_nov, 0) AS delta_nov
        , ISNULL(d.delta_pr2_pr3, 0) AS delta_pr2_pr3
        , ISNULL(d.delta_ndp_ndm, 0) AS delta_ndp_ndm
        , ISNULL(d.delta_pk2_ot1, 0) AS delta_pk2_ot1
        , ISNULL(d.delta_pk3_pk6, 0) AS delta_pk3_pk6
        , ISNULL(d.delta_mpl, 0) AS delta_mpl
        , ISNULL(d.delta_pns, 0) AS delta_pns
        , ISNULL(d.delta_other, 0) AS delta_other

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
              ROWS BETWEEN UNBOUNDED PRECEDING
                       AND CURRENT ROW
          ) AS balance_float


        /* 2. ФУ */
        , SUM(delta_fu) OVER
          (
              ORDER BY dt_rep
              ROWS BETWEEN UNBOUNDED PRECEDING
                       AND CURRENT ROW
          ) AS balance_fu


        /* 3. Нов */
        , SUM(delta_nov) OVER
          (
              ORDER BY dt_rep
              ROWS BETWEEN UNBOUNDED PRECEDING
                       AND CURRENT ROW
          ) AS balance_nov


        /* 4. Пр2 / Пр3 */
        , SUM(delta_pr2_pr3) OVER
          (
              ORDER BY dt_rep
              ROWS BETWEEN UNBOUNDED PRECEDING
                       AND CURRENT ROW
          ) AS balance_pr2_pr3


        /* 5. НДП / НДМ */
        , SUM(delta_ndp_ndm) OVER
          (
              ORDER BY dt_rep
              ROWS BETWEEN UNBOUNDED PRECEDING
                       AND CURRENT ROW
          ) AS balance_ndp_ndm


        /* 6. Пк2 / От1 */
        , SUM(delta_pk2_ot1) OVER
          (
              ORDER BY dt_rep
              ROWS BETWEEN UNBOUNDED PRECEDING
                       AND CURRENT ROW
          ) AS balance_pk2_ot1


        /* 7. Пк3 / Пк6 */
        , SUM(delta_pk3_pk6) OVER
          (
              ORDER BY dt_rep
              ROWS BETWEEN UNBOUNDED PRECEDING
                       AND CURRENT ROW
          ) AS balance_pk3_pk6


        /* 8. МПЛ */
        , SUM(delta_mpl) OVER
          (
              ORDER BY dt_rep
              ROWS BETWEEN UNBOUNDED PRECEDING
                       AND CURRENT ROW
          ) AS balance_mpl


        /* 9. Пнс */
        , SUM(delta_pns) OVER
          (
              ORDER BY dt_rep
              ROWS BETWEEN UNBOUNDED PRECEDING
                       AND CURRENT ROW
          ) AS balance_pns


        /* 10. Остальные */
        , SUM(delta_other) OVER
          (
              ORDER BY dt_rep
              ROWS BETWEEN UNBOUNDED PRECEDING
                       AND CURRENT ROW
          ) AS balance_other


    FROM daily
)


/* ============================================================
   6. РЕЗУЛЬТАТ

   Одна строка = одна календарная дата.

   Все вклады полностью разложены по взаимоисключающим
   категориям.
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


    /* Полный остаток */
    , (
          balance_float
        + balance_fu
        + balance_nov
        + balance_pr2_pr3
        + balance_ndp_ndm
        + balance_pk2_ot1
        + balance_pk3_pk6
        + balance_mpl
        + balance_pns
        + balance_other
      ) AS balance_total


FROM balances

ORDER BY
    dt_rep

OPTION (MAXRECURSION 0, RECOMPILE);
