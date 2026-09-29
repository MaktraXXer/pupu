/* ============================================================
   ПРОВЕРКА:
   ВСЕ ФАКТИЧЕСКИ ЖИВЫЕ ВКЛАДЫ КЛИЕНТОВ НА 26.09.2026

   Живой вклад = по CON_ID существует ненулевой OUT_RUB
   в DepositContract_Saldo, действующий на @CheckDate.

   #clients уже создан предыдущим скриптом.
   ============================================================ */

DECLARE @CheckDate date = '2026-09-26';


/* ------------------------------------------------------------
   1. Находим все CON_ID с фактическим остатком на дату
   ------------------------------------------------------------ */

DROP TABLE IF EXISTS #live_deposits_2609;

SELECT
      s.CON_ID AS con_id

    /* фактический остаток вклада на дату */
    , SUM(
          CAST(s.OUT_RUB AS decimal(38,6))
      ) AS balance_rub_2609

INTO #live_deposits_2609

FROM [LIQUIDITY].[liq].[DepositContract_Saldo] s WITH (NOLOCK)

WHERE
    s.DT_FROM <= @CheckDate

    AND
    (
        s.DT_TO IS NULL
        OR s.DT_TO >= @CheckDate
    )

    AND s.OUT_RUB IS NOT NULL

GROUP BY
    s.CON_ID

HAVING
    SUM(
        CAST(s.OUT_RUB AS decimal(38,6))
    ) <> 0;


CREATE UNIQUE CLUSTERED INDEX IX_live_deposits_2609
ON #live_deposits_2609 (con_id);



/* ------------------------------------------------------------
   2. Оставляем только вклады наших 3 клиентов

   Для каждого CON_ID берём последнюю известную запись SNAP,
   чтобы вывести ВСЕ атрибуты договора.

   Факт "живой / не живой" определяется НЕ SNAP,
   а наличием фактического сальдо выше.
   ------------------------------------------------------------ */

;WITH snap_ranked AS
(
    SELECT
          d.*

        , ROW_NUMBER() OVER
          (
              PARTITION BY d.CON_ID

              ORDER BY
                    d.DT_REP DESC
          ) AS rn_live_check

    FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap] d WITH (NOLOCK)

    INNER JOIN #clients c
        ON c.cli_id = d.CLI_ID

    INNER JOIN #live_deposits_2609 l
        ON l.con_id = d.CON_ID

    WHERE
        d.[TSEGMENTNAME] = N'Розничный бизнес'
)

SELECT
      s.*

    /* отдельно добавляем фактический остаток именно на 26.09 */
    , l.balance_rub_2609

FROM snap_ranked s

INNER JOIN #live_deposits_2609 l
    ON l.con_id = s.CON_ID

WHERE
    s.rn_live_check = 1

ORDER BY
      s.CLI_ID
    , s.CUR
    , s.CON_ID;
