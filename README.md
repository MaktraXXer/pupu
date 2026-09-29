/* ============================================================
   ВСЕ ВКЛАДЫ, КОТОРЫЕ ФОРМИРУЮТ DEPOSIT_BALANCE
   НА 26.09.2026 В ОСНОВНОМ РАСЧЁТЕ

   Используем непосредственно #saldo из основного скрипта,
   поэтому сумма этих строк должна совпасть
   с deposit_balance первого Result Set.
   ============================================================ */

DECLARE @CheckDate2 date = '2026-09-26';


/* ============================================================
   1. Сначала схлопываем сальдо до одного CON_ID
   ============================================================ */

DROP TABLE IF EXISTS #live_dep_check;

SELECT
      s.cli_id
    , s.cur
    , s.con_id

    , SUM(
          CAST(s.balance_rub AS decimal(38,6))
      ) AS balance_rub

INTO #live_dep_check

FROM #saldo s

WHERE
    s.contract_type = 'DEP'

    AND @CheckDate2
        BETWEEN s.effective_from
            AND s.effective_to

GROUP BY
      s.cli_id
    , s.cur
    , s.con_id

HAVING
    SUM(
        CAST(s.balance_rub AS decimal(38,6))
    ) <> 0;


CREATE UNIQUE CLUSTERED INDEX IX_live_dep_check
ON #live_dep_check (con_id);



/* ============================================================
   2. ВЫВОДИМ ВСЕ ЖИВЫЕ ВКЛАДЫ

   Берём атрибуты из последнего доступного SNAP
   по каждому найденному CON_ID.

   ВАЖНО:
   OUTER APPLY, а не CROSS APPLY.

   Поэтому даже если для CON_ID почему-либо не найдётся
   строка SNAP, сам CON_ID всё равно будет показан.
   ============================================================ */

SELECT
      l.cli_id
    , l.cur
    , l.con_id

    /* Остаток, который реально вошёл
       в deposit_balance на 26.09 */
    , l.balance_rub AS balance_rub_2609


    /* Поля договора */
    , dc.dt_open
    , dc.dt_close
    , dc.dt_close_plan
    , dc.fallback_rate


    /* Поля последнего SNAP */
    , snap.DT_REP          AS snap_dt_rep
    , snap.RATE            AS snap_rate
    , snap.CUR             AS snap_cur
    , snap.BALANCE_RUB     AS snap_balance_rub
    , snap.PROD_NAME       AS prod_name
    , snap.TSEGMENTNAME    AS tsegmentname


FROM #live_dep_check l


/* Реестр договора из основного расчёта */
LEFT JOIN #deposit_contracts dc
    ON dc.con_id = l.con_id


/* Последняя найденная запись SNAP */
OUTER APPLY
(
    SELECT TOP (1)
          d.DT_REP
        , d.RATE
        , d.CUR
        , d.BALANCE_RUB
        , d.PROD_NAME
        , d.TSEGMENTNAME

    FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap] d WITH (NOLOCK)

    WHERE
        d.CON_ID = l.con_id

    ORDER BY
        d.DT_REP DESC

) snap


ORDER BY
      l.cli_id
    , l.cur
    , l.balance_rub DESC
    , l.con_id;



/* ============================================================
   3. КОНТРОЛЬНАЯ СУММА

   Должна совпасть с DEPOSIT_BALANCE
   первого Result Set на 26.09
   ============================================================ */

SELECT
      cli_id
    , cur

    , COUNT(*) AS deposit_count

    , SUM(balance_rub) AS deposit_balance_check

FROM #live_dep_check

GROUP BY
      cli_id
    , cur

ORDER BY
      cli_id
    , cur;
