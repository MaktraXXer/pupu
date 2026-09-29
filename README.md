USE [ALM_TEST];
SET NOCOUNT ON;


/* ============================================================
   0. ПАРАМЕТРЫ
   ============================================================ */

DECLARE @DateFrom date = '2024-01-01';
DECLARE @DateTo   date = '2026-09-26';


/* ============================================================
   1. КЛИЕНТЫ

   МЕНЯТЬ ТОЛЬКО ЭТИ ТРИ CLI_ID
   ============================================================ */

DROP TABLE IF EXISTS #clients;

CREATE TABLE #clients
(
    cli_id bigint NOT NULL
        PRIMARY KEY
);

INSERT INTO #clients (cli_id)
VALUES
      (1111111111)       -- CLI_ID 1
    , (2222222222)       -- CLI_ID 2
    , (3333333333);      -- CLI_ID 3



/* ============================================================
   2. КАЛЕНДАРЬ
   ============================================================ */

DROP TABLE IF EXISTS #calendar;

;WITH calendar AS
(
    SELECT
        @DateFrom AS dt_rep

    UNION ALL

    SELECT
        DATEADD(day, 1, dt_rep)

    FROM calendar

    WHERE dt_rep < @DateTo
)

SELECT
    dt_rep

INTO #calendar

FROM calendar

OPTION (MAXRECURSION 0);


CREATE UNIQUE CLUSTERED INDEX IX_calendar
ON #calendar (dt_rep);



/* ============================================================
   3. ВКЛАДЫ

   DepositInterestsRateSnap используем как РЕЕСТР договоров.

   Для каждого CON_ID берём последнюю известную запись,
   чтобы получить:

   - CLI_ID
   - DT_OPEN
   - DT_CLOSE
   - DT_CLOSE_PLAN
   - RATE
   - CUR

   DT_REP снапшота НЕ используется как период действия остатка.
   ============================================================ */

DROP TABLE IF EXISTS #deposit_contracts;

;WITH deposit_ranked AS
(
    SELECT
          CAST(d.CON_ID AS bigint) AS con_id
        , CAST(d.CLI_ID AS bigint) AS cli_id

        , CAST(d.DT_OPEN AS date) AS dt_open
        , CAST(d.DT_CLOSE AS date) AS dt_close
        , CAST(d.DT_CLOSE_PLAN AS date) AS dt_close_plan

        , TRY_CAST(d.RATE AS decimal(18,10)) AS rate

        /* Нормализуем обозначение рублей */
        , CASE
              WHEN d.CUR IS NULL
                  THEN 'UNKNOWN'

              WHEN UPPER(LTRIM(RTRIM(CAST(d.CUR AS varchar(20)))))
                   IN ('810', '643', 'RUR', 'RUB')
                  THEN 'RUR'

              ELSE
                  UPPER(LTRIM(RTRIM(CAST(d.CUR AS varchar(20)))))
          END AS cur

        , ROW_NUMBER() OVER
          (
              PARTITION BY d.CON_ID

              ORDER BY
                  d.DT_REP DESC
          ) AS rn

    FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap] d WITH (NOLOCK)

    INNER JOIN #clients c
        ON c.cli_id = d.CLI_ID

    WHERE
        d.DT_OPEN <= @DateTo

        AND d.[TSEGMENTNAME] = N'Розничный бизнес'
)

SELECT
      con_id
    , cli_id

    , dt_open
    , dt_close
    , dt_close_plan

    , rate
    , cur

INTO #deposit_contracts

FROM deposit_ranked

WHERE
    rn = 1

    /* Договор пересекает исследуемый период */
    AND dt_open <= @DateTo

    AND
    (
        dt_close IS NULL
        OR dt_close >= @DateFrom
    );


CREATE UNIQUE CLUSTERED INDEX IX_deposit_contracts_con
ON #deposit_contracts (con_id);



/* ============================================================
   4. НАКОПИТЕЛЬНЫЕ СЧЕТА

   Отбираем только нужных клиентов и два нужных продукта.

   На каждый CON_ID оставляем последнюю известную запись.
   ============================================================ */

DROP TABLE IF EXISTS #ns_contracts;

;WITH ns_ranked AS
(
    SELECT
          CAST(n.CON_ID AS bigint) AS con_id
        , CAST(n.CLI_ID AS bigint) AS cli_id

        , CAST(n.DT_OPEN AS date) AS dt_open
        , CAST(n.DT_CLOSE AS date) AS dt_close
        , CAST(n.DT_CLOSE_PLAN AS date) AS dt_close_plan

        , TRY_CAST(n.RATE AS decimal(18,10)) AS rate

        , CASE
              WHEN n.CUR IS NULL
                  THEN 'UNKNOWN'

              WHEN UPPER(LTRIM(RTRIM(CAST(n.CUR AS varchar(20)))))
                   IN ('810', '643', 'RUR', 'RUB')
                  THEN 'RUR'

              ELSE
                  UPPER(LTRIM(RTRIM(CAST(n.CUR AS varchar(20)))))
          END AS cur

        , ROW_NUMBER() OVER
          (
              PARTITION BY n.CON_ID

              ORDER BY
                    n.DT_UPDATE DESC
                  , n.DT_IMPORT DESC
          ) AS rn

    FROM [LIQUIDITY].[liq].[depositcontract_all] n WITH (NOLOCK)

    INNER JOIN #clients c
        ON c.cli_id = n.CLI_ID

    WHERE
        n.CLI_SHORT_NAME = N'ФЛ'

        AND n.PROD_NAME IN
        (
              N'Накопительный счёт Ультра'
            , N'Накопительный счёт'
        )

        AND n.DT_OPEN <= @DateTo
)

SELECT
      con_id
    , cli_id

    , dt_open
    , dt_close
    , dt_close_plan

    , rate
    , cur

INTO #ns_contracts

FROM ns_ranked

WHERE
    rn = 1

    AND dt_open <= @DateTo

    AND
    (
        dt_close IS NULL
        OR dt_close >= @DateFrom
    );


CREATE UNIQUE CLUSTERED INDEX IX_ns_contracts_con
ON #ns_contracts (con_id);



/* ============================================================
   5. ЗАЩИТА ОТ ДВОЙНОГО УЧЁТА

   Если какой-то НС вдруг также присутствует
   в DepositInterestsRateSnap, считаем его именно НС.
   ============================================================ */

DELETE d

FROM #deposit_contracts d

INNER JOIN #ns_contracts n
    ON n.con_id = d.con_id;



/* ============================================================
   6. ОБЪЕДИНЯЕМ НУЖНЫЕ CON_ID

   Только после этого пойдём в тяжёлое сальдо.
   ============================================================ */

DROP TABLE IF EXISTS #all_contracts;

SELECT
      d.con_id
    , d.cli_id

    , d.dt_open
    , d.dt_close
    , d.dt_close_plan

    , d.rate
    , d.cur

    , CAST('DEP' AS varchar(3)) AS contract_type

INTO #all_contracts

FROM #deposit_contracts d


UNION ALL


SELECT
      n.con_id
    , n.cli_id

    , n.dt_open
    , n.dt_close
    , n.dt_close_plan

    , n.rate
    , n.cur

    , CAST('NS' AS varchar(3)) AS contract_type

FROM #ns_contracts n;


CREATE UNIQUE CLUSTERED INDEX IX_all_contracts
ON #all_contracts (con_id, contract_type);



/* ============================================================
   7. САЛЬДО

   ВАЖНО:
   DepositContract_Saldo читаем ТОЛЬКО для уже отобранных CON_ID.

   В исходной таблице фактический рублёвый остаток = OUT_RUB.

   Здесь сразу переименовываем его логически в balance_rub.

   effective_from =
       MAX(
           DT_FROM сальдо,
           DT_OPEN договора,
           @DateFrom
       )

   effective_to =
       MIN(
           DT_TO сальдо,
           DT_CLOSE договора,
           @DateTo
       )
   ============================================================ */

DROP TABLE IF EXISTS #saldo;

SELECT
      ac.con_id
    , ac.cli_id
    , ac.contract_type

    , ac.cur
    , ac.rate

    , bounds_from.effective_from
    , bounds_to.effective_to

    , CAST(s.OUT_RUB AS decimal(38,6)) AS balance_rub

INTO #saldo

FROM [LIQUIDITY].[liq].[DepositContract_Saldo] s WITH (NOLOCK)

INNER JOIN #all_contracts ac
    ON ac.con_id = s.CON_ID


CROSS APPLY
(
    SELECT
        MAX(v.dt) AS effective_from

    FROM
    (
        VALUES
              (CAST(s.DT_FROM AS date))
            , (ac.dt_open)
            , (@DateFrom)
    ) v(dt)

) bounds_from


CROSS APPLY
(
    SELECT
        MIN(v.dt) AS effective_to

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
                      ac.dt_close,
                      @DateTo
                  )
              )

            , (@DateTo)

    ) v(dt)

) bounds_to


WHERE
    /* Сальдо пересекает наш горизонт */
    s.DT_FROM <= @DateTo

    AND
    (
        s.DT_TO IS NULL
        OR s.DT_TO >= @DateFrom
    )

    AND s.OUT_RUB IS NOT NULL

    /* После всех ограничений интервал существует */
    AND bounds_from.effective_from
        <= bounds_to.effective_to;


CREATE CLUSTERED INDEX IX_saldo
ON #saldo
(
      cli_id
    , cur
    , effective_from
    , effective_to
);



/* ============================================================
   8. ПОДНЕВНАЯ АГРЕГАЦИЯ

   По каждой дате:
   - остаток вкладов
   - остаток НС
   - средневзвешенная ставка вкладов
   - средневзвешенная ставка НС

   Средневзвешенная ставка:

       SUM(balance_rub * rate)
       -----------------------
           SUM(balance_rub)

   Если RATE = NULL, такой договор не участвует
   в расчёте средней ставки.
   ============================================================ */

DROP TABLE IF EXISTS #daily_agg;

SELECT
      c.dt_rep
    , s.cli_id
    , s.cur


    /* -------------------------
       Остаток вкладов
       ------------------------- */

    , SUM(
          CASE
              WHEN s.contract_type = 'DEP'
                  THEN s.balance_rub
              ELSE 0
          END
      ) AS deposit_balance


    /* -------------------------
       Остаток НС
       ------------------------- */

    , SUM(
          CASE
              WHEN s.contract_type = 'NS'
                  THEN s.balance_rub
              ELSE 0
          END
      ) AS ns_balance


    /* -------------------------
       Ставка вкладов
       ------------------------- */

    , CAST(

        SUM(
            CASE
                WHEN s.contract_type = 'DEP'
                     AND s.rate IS NOT NULL

                    THEN s.balance_rub * s.rate

                ELSE 0
            END
        )

        /

        NULLIF(
            SUM(
                CASE
                    WHEN s.contract_type = 'DEP'
                         AND s.rate IS NOT NULL

                        THEN s.balance_rub

                    ELSE 0
                END
            ),
            0
        )

      AS decimal(18,10)) AS deposit_rate


    /* -------------------------
       Ставка НС
       ------------------------- */

    , CAST(

        SUM(
            CASE
                WHEN s.contract_type = 'NS'
                     AND s.rate IS NOT NULL

                    THEN s.balance_rub * s.rate

                ELSE 0
            END
        )

        /

        NULLIF(
            SUM(
                CASE
                    WHEN s.contract_type = 'NS'
                         AND s.rate IS NOT NULL

                        THEN s.balance_rub

                    ELSE 0
                END
            ),
            0
        )

      AS decimal(18,10)) AS ns_rate


INTO #daily_agg

FROM #calendar c

INNER JOIN #saldo s
    ON c.dt_rep
       BETWEEN s.effective_from
           AND s.effective_to

GROUP BY
      c.dt_rep
    , s.cli_id
    , s.cur;


CREATE UNIQUE CLUSTERED INDEX IX_daily_agg
ON #daily_agg
(
      dt_rep
    , cli_id
    , cur
);



/* ============================================================
   9. ФЛАГ ДОСРОЧНОГО ЗАКРЫТИЯ ВКЛАДА

   Только ВКЛАДЫ.

   flag = 1, если на эту дату у клиента есть договор:

       DT_CLOSE = DT_REP

   и одновременно

       DT_CLOSE <> DT_CLOSE_PLAN

   Флаг считается на уровне CLIENT + DATE,
   то есть если клиент досрочно закрыл вклад в этот день,
   flag = 1 для его строки/строк этого дня.
   ============================================================ */

DROP TABLE IF EXISTS #early_close;

SELECT
      d.cli_id
    , d.dt_close AS dt_rep
    , CAST(1 AS tinyint) AS early_close_flag

INTO #early_close

FROM #deposit_contracts d

WHERE
    d.dt_close BETWEEN @DateFrom AND @DateTo

    AND d.dt_close_plan IS NOT NULL

    AND d.dt_close <> d.dt_close_plan

GROUP BY
      d.cli_id
    , d.dt_close;


CREATE UNIQUE CLUSTERED INDEX IX_early_close
ON #early_close
(
      dt_rep
    , cli_id
);



/* ============================================================
   10. СПИСОК ВАЛЮТ КАЖДОГО КЛИЕНТА

   Нужен, чтобы получить ПОЛНОЕ подневное полотно,
   включая дни с нулевым остатком.
   ============================================================ */

DROP TABLE IF EXISTS #client_currency;

SELECT DISTINCT
      cli_id
    , cur

INTO #client_currency

FROM #all_contracts;


CREATE UNIQUE CLUSTERED INDEX IX_client_currency
ON #client_currency
(
      cli_id
    , cur
);



/* ============================================================
   11. ФИНАЛЬНЫЙ РЕЗУЛЬТАТ

   Одна строка:

       DT_REP
       CLI_ID
       CUR

       DEPOSIT_BALANCE
       NS_BALANCE

       DEPOSIT_RATE
       NS_RATE

       EARLY_CLOSE_FLAG

   ============================================================ */

SELECT
      c.dt_rep

    , cc.cli_id
    , cc.cur


    /* Остатки */
    , ISNULL(
          d.deposit_balance,
          CAST(0 AS decimal(38,6))
      ) AS deposit_balance

    , ISNULL(
          d.ns_balance,
          CAST(0 AS decimal(38,6))
      ) AS ns_balance


    /* Ставки.
       Если соответствующего остатка нет -> NULL */
    , d.deposit_rate
    , d.ns_rate


    /* Досрочное закрытие вклада */
    , ISNULL(
          e.early_close_flag,
          0
      ) AS early_close_flag


FROM #calendar c

CROSS JOIN #client_currency cc


LEFT JOIN #daily_agg d
    ON  d.dt_rep = c.dt_rep
    AND d.cli_id = cc.cli_id
    AND d.cur    = cc.cur


LEFT JOIN #early_close e
    ON  e.dt_rep = c.dt_rep
    AND e.cli_id = cc.cli_id


ORDER BY
      c.dt_rep
    , cc.cli_id
    , cc.cur

OPTION (RECOMPILE);
