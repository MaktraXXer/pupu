USE [ALM_TEST];
SET NOCOUNT ON;


/* ============================================================
   0. ПАРАМЕТРЫ
   ============================================================ */

DECLARE @DateFrom      date = '2024-01-01';
DECLARE @DateTo        date = '2026-09-26';

/* Нужно для определения остатка за день до досрочного закрытия */
DECLARE @SaldoDateFrom date = DATEADD(day, -1, @DateFrom);



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
      (1111111111)      -- CLI_ID 1
    , (2222222222)      -- CLI_ID 2
    , (3333333333);     -- CLI_ID 3



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
   3. РЕЕСТР ВКЛАДОВ

   DepositInterestsRateSnap используем для определения:
   - CON_ID
   - CLI_ID
   - DT_OPEN
   - DT_CLOSE
   - DT_CLOSE_PLAN
   - CUR
   - RATE

   Здесь на договор берём последнюю известную запись
   только для получения атрибутов самого договора.

   Историческую ставку дальше считаем отдельно по DT_REP.
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

        , TRY_CAST(d.RATE AS decimal(18,10)) AS fallback_rate

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
    , fallback_rate
    , cur

INTO #deposit_contracts

FROM deposit_ranked

WHERE
    rn = 1

    AND dt_open <= @DateTo

    AND
    (
        dt_close IS NULL
        OR dt_close >= @DateFrom
    );


CREATE UNIQUE CLUSTERED INDEX IX_deposit_contracts
ON #deposit_contracts (con_id);



/* ============================================================
   4. РЕЕСТР НАКОПИТЕЛЬНЫХ СЧЕТОВ

   Только:
   - Накопительный счёт
   - Накопительный счёт Ультра

   На каждый CON_ID берём последнюю запись.
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


CREATE UNIQUE CLUSTERED INDEX IX_ns_contracts
ON #ns_contracts (con_id);



/* ============================================================
   5. ЗАЩИТА ОТ ДВОЙНОГО УЧЁТА

   Если НС одновременно присутствует в DepositInterestsRateSnap,
   считаем этот договор именно НС.
   ============================================================ */

DELETE d

FROM #deposit_contracts d

INNER JOIN #ns_contracts n
    ON n.con_id = d.con_id;



/* ============================================================
   6. ИСТОРИЯ СТАВОК ПО ВКЛАДАМ

   RATE берётся из DepositInterestsRateSnap.

   Одна строка:
       CON_ID + DATE

   Если в одну дату несколько записей,
   оставляем последнюю.
   ============================================================ */

DROP TABLE IF EXISTS #deposit_rates;

;WITH rate_ranked AS
(
    SELECT
          dc.con_id

        , CAST(d.DT_REP AS date) AS rate_date

        , TRY_CAST(
              d.RATE AS decimal(18,10)
          ) AS rate

        , ROW_NUMBER() OVER
          (
              PARTITION BY
                    dc.con_id
                  , CAST(d.DT_REP AS date)

              ORDER BY
                  d.DT_REP DESC
          ) AS rn

    FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap] d WITH (NOLOCK)

    INNER JOIN #deposit_contracts dc
        ON dc.con_id = d.CON_ID

    WHERE
        d.DT_REP IS NOT NULL

        AND CAST(d.DT_REP AS date) <= @DateTo
)

SELECT
      con_id
    , rate_date
    , rate

INTO #deposit_rates

FROM rate_ranked

WHERE rn = 1;


CREATE UNIQUE CLUSTERED INDEX IX_deposit_rates
ON #deposit_rates
(
      con_id
    , rate_date
);



/* ============================================================
   7. ПРЕВРАЩАЕМ СНАПШОТЫ СТАВОК В ИНТЕРВАЛЫ

   Например:

       01.01 ставка 15%
       10.01 ставка 16%

   превращается в:

       01.01 - 09.01 = 15%
       10.01 - ...   = 16%
   ============================================================ */

DROP TABLE IF EXISTS #deposit_rate_intervals;

;WITH rate_lead AS
(
    SELECT
          r.con_id
        , r.rate_date
        , r.rate

        , LEAD(r.rate_date) OVER
          (
              PARTITION BY r.con_id
              ORDER BY r.rate_date
          ) AS next_rate_date

    FROM #deposit_rates r
),

rate_bounds AS
(
    SELECT
          r.con_id
        , r.rate

        , bf.rate_from
        , bt.rate_to

    FROM rate_lead r

    INNER JOIN #deposit_contracts dc
        ON dc.con_id = r.con_id

    CROSS APPLY
    (
        SELECT
            MAX(v.dt) AS rate_from

        FROM
        (
            VALUES
                  (r.rate_date)
                , (dc.dt_open)
                , (@SaldoDateFrom)
        ) v(dt)

    ) bf

    CROSS APPLY
    (
        SELECT
            MIN(v.dt) AS rate_to

        FROM
        (
            VALUES
                  (
                      ISNULL(
                          DATEADD(day, -1, r.next_rate_date),
                          @DateTo
                      )
                  )

                , (
                      ISNULL(
                          dc.dt_close,
                          @DateTo
                      )
                  )

                , (@DateTo)

        ) v(dt)

    ) bt
)

SELECT
      con_id
    , rate
    , rate_from
    , rate_to

INTO #deposit_rate_intervals

FROM rate_bounds

WHERE
    rate_from <= rate_to;



/* Если почему-либо по договору вообще нет исторического
   RATE в снапшоте, используем RATE из последней записи договора. */

INSERT INTO #deposit_rate_intervals
(
      con_id
    , rate
    , rate_from
    , rate_to
)

SELECT
      dc.con_id
    , dc.fallback_rate

    , bf.rate_from
    , bt.rate_to

FROM #deposit_contracts dc

CROSS APPLY
(
    SELECT
        MAX(v.dt) AS rate_from

    FROM
    (
        VALUES
              (dc.dt_open)
            , (@SaldoDateFrom)
    ) v(dt)

) bf

CROSS APPLY
(
    SELECT
        MIN(v.dt) AS rate_to

    FROM
    (
        VALUES
              (ISNULL(dc.dt_close, @DateTo))
            , (@DateTo)
    ) v(dt)

) bt

WHERE
    dc.fallback_rate IS NOT NULL

    AND bf.rate_from <= bt.rate_to

    AND NOT EXISTS
    (
        SELECT 1

        FROM #deposit_rates r

        WHERE r.con_id = dc.con_id
    );


CREATE CLUSTERED INDEX IX_deposit_rate_intervals
ON #deposit_rate_intervals
(
      con_id
    , rate_from
    , rate_to
);



/* ============================================================
   8. ОБЪЕДИНЯЕМ ВСЕ НУЖНЫЕ CON_ID

   После этого тяжёлое сальдо читается ТОЛЬКО
   для этих договоров.
   ============================================================ */

DROP TABLE IF EXISTS #all_contracts;

SELECT
      d.con_id
    , d.cli_id

    , d.dt_open
    , d.dt_close
    , d.dt_close_plan

    , d.fallback_rate AS base_rate
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

    , n.rate AS base_rate
    , n.cur

    , CAST('NS' AS varchar(3)) AS contract_type

FROM #ns_contracts n;


CREATE UNIQUE CLUSTERED INDEX IX_all_contracts
ON #all_contracts
(
      con_id
    , contract_type
);



/* ============================================================
   9. ФАКТИЧЕСКОЕ САЛЬДО

   DepositContract_Saldo:

       DT_FROM ... DT_TO = OUT_RUB

   Читаем ТОЛЬКО CON_ID из #all_contracts.

   balance_rub = OUT_RUB

   Интервал ограничиваем:
   - периодом сальдо
   - жизнью договора
   - горизонтом расчёта

   @SaldoDateFrom = 31.12.2023,
   чтобы можно было определить остаток за день до досрочки
   01.01.2024.
   ============================================================ */

DROP TABLE IF EXISTS #saldo;

SELECT
      ac.con_id
    , ac.cli_id
    , ac.contract_type

    , ac.cur
    , ac.base_rate

    , bf.effective_from
    , bt.effective_to

    , CAST(
          s.OUT_RUB AS decimal(38,6)
      ) AS balance_rub

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
            , (@SaldoDateFrom)
    ) v(dt)

) bf


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

) bt


WHERE
    s.DT_FROM <= @DateTo

    AND
    (
        s.DT_TO IS NULL
        OR s.DT_TO >= @SaldoDateFrom
    )

    AND s.OUT_RUB IS NOT NULL

    AND bf.effective_from <= bt.effective_to;


CREATE CLUSTERED INDEX IX_saldo_client
ON #saldo
(
      cli_id
    , cur
    , effective_from
    , effective_to
);


CREATE NONCLUSTERED INDEX IX_saldo_contract
ON #saldo
(
      con_id
    , contract_type
    , effective_from
    , effective_to
)

INCLUDE
(
      balance_rub
    , base_rate
    , cli_id
    , cur
);



/* ============================================================
   10. ПОДНЕВНАЯ АГРЕГАЦИЯ

   На каждый:
       DT_REP + CLI_ID + CUR

   считаем:

   - остаток вкладов
   - остаток НС
   - средневзвешенную ставку вкладов
   - средневзвешенную ставку НС

   ВЕС = balance_rub

   Для вкладов:
       RATE берём исторический на соответствующую дату.

   Если RATE в истории почему-либо отсутствует:
       используем fallback RATE договора.

   Для НС:
       используем RATE договора.
   ============================================================ */

DROP TABLE IF EXISTS #daily_agg;

SELECT
      c.dt_rep
    , s.cli_id
    , s.cur


    /* ========================================================
       ОСТАТОК ВКЛАДОВ
       ======================================================== */

    , SUM(
          CASE
              WHEN s.contract_type = 'DEP'
                  THEN s.balance_rub

              ELSE 0
          END
      ) AS deposit_balance


    /* ========================================================
       ОСТАТОК НС
       ======================================================== */

    , SUM(
          CASE
              WHEN s.contract_type = 'NS'
                  THEN s.balance_rub

              ELSE 0
          END
      ) AS ns_balance


    /* ========================================================
       СРЕДНЕВЗВЕШЕННАЯ СТАВКА ВКЛАДОВ
       ======================================================== */

    , CAST(

          SUM(
              CASE
                  WHEN s.contract_type = 'DEP'
                       AND er.effective_rate IS NOT NULL

                      THEN
                          s.balance_rub
                          * er.effective_rate

                  ELSE 0
              END
          )

          /

          NULLIF(
              SUM(
                  CASE
                      WHEN s.contract_type = 'DEP'
                           AND er.effective_rate IS NOT NULL

                          THEN s.balance_rub

                      ELSE 0
                  END
              ),
              0
          )

      AS decimal(18,10)) AS deposit_rate


    /* ========================================================
       СРЕДНЕВЗВЕШЕННАЯ СТАВКА НС
       ======================================================== */

    , CAST(

          SUM(
              CASE
                  WHEN s.contract_type = 'NS'
                       AND s.base_rate IS NOT NULL

                      THEN
                          s.balance_rub
                          * s.base_rate

                  ELSE 0
              END
          )

          /

          NULLIF(
              SUM(
                  CASE
                      WHEN s.contract_type = 'NS'
                           AND s.base_rate IS NOT NULL

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


/* Историческая ставка для вклада */
LEFT JOIN #deposit_rate_intervals dri
    ON  s.contract_type = 'DEP'
    AND dri.con_id = s.con_id

    AND c.dt_rep
        BETWEEN dri.rate_from
            AND dri.rate_to


CROSS APPLY
(
    SELECT
        CASE
            WHEN s.contract_type = 'DEP'
                THEN COALESCE(
                         dri.rate,
                         s.base_rate
                     )

            ELSE s.base_rate
        END AS effective_rate

) er


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
   11. ДОСРОЧНЫЕ ЗАКРЫТИЯ — ПО КАЖДОМУ ВКЛАДУ

   Досрочка:

       DT_CLOSE <> DT_CLOSE_PLAN

   Для каждого закрытого вклада определяем:

   - остаток за ДЕНЬ ДО закрытия
   - ставку за ДЕНЬ ДО закрытия
   ============================================================ */

DROP TABLE IF EXISTS #early_close_detail;

SELECT
      d.cli_id
    , d.cur
    , d.con_id

    , d.dt_close AS dt_rep

    /* Ставка за день до досрочного закрытия */
    , MAX(
          COALESCE(
              dri.rate,
              d.fallback_rate
          )
      ) AS rate_before_close


    /* Остаток на конец дня перед закрытием */
    , ISNULL(
          SUM(s.balance_rub),
          CAST(0 AS decimal(38,6))
      ) AS balance_before_close


INTO #early_close_detail

FROM #deposit_contracts d


/* Остаток за предыдущий день */
LEFT JOIN #saldo s
    ON  s.con_id = d.con_id
    AND s.contract_type = 'DEP'

    AND DATEADD(day, -1, d.dt_close)
        BETWEEN s.effective_from
            AND s.effective_to


/* Ставка за предыдущий день */
LEFT JOIN #deposit_rate_intervals dri
    ON dri.con_id = d.con_id

    AND DATEADD(day, -1, d.dt_close)
        BETWEEN dri.rate_from
            AND dri.rate_to


WHERE
    d.dt_close BETWEEN @DateFrom AND @DateTo

    AND d.dt_close_plan IS NOT NULL

    AND d.dt_close <> d.dt_close_plan


GROUP BY
      d.cli_id
    , d.cur
    , d.con_id
    , d.dt_close;


CREATE UNIQUE CLUSTERED INDEX IX_early_close_detail
ON #early_close_detail
(
      dt_rep
    , cli_id
    , cur
    , con_id
);



/* ============================================================
   12. ДОСРОЧКИ — АГРЕГАЦИЯ ПО КЛИЕНТУ / ВАЛЮТЕ / ДАТЕ

   Получаем:

   early_close_flag
       = была ли досрочка

   early_close_count
       = сколько вкладов закрыто досрочно

   early_close_balance_rub
       = сколько рублей было на этих вкладах
         за день до закрытия

   early_close_rate
       = средневзвешенная ставка этих вкладов
         за день до закрытия
   ============================================================ */

DROP TABLE IF EXISTS #early_close;

SELECT
      cli_id
    , cur
    , dt_rep

    , CAST(1 AS tinyint) AS early_close_flag


    /* Количество вкладов */
    , COUNT(*) AS early_close_count


    /* Объём досрочно закрытых вкладов */
    , SUM(
          balance_before_close
      ) AS early_close_balance_rub


    /* Средневзвешенная ставка */
    , CAST(

          SUM(
              CASE
                  WHEN rate_before_close IS NOT NULL
                      THEN
                          balance_before_close
                          * rate_before_close

                  ELSE 0
              END
          )

          /

          NULLIF(
              SUM(
                  CASE
                      WHEN rate_before_close IS NOT NULL
                          THEN balance_before_close

                      ELSE 0
                  END
              ),
              0
          )

      AS decimal(18,10)) AS early_close_rate


INTO #early_close

FROM #early_close_detail

GROUP BY
      cli_id
    , cur
    , dt_rep;


CREATE UNIQUE CLUSTERED INDEX IX_early_close
ON #early_close
(
      dt_rep
    , cli_id
    , cur
);



/* ============================================================
   13. ВАЛЮТЫ КЛИЕНТОВ

   Нужны для полного ежедневного полотна,
   в том числе когда остаток в конкретный день = 0.
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
   14. ОСНОВНОЙ РЕЗУЛЬТАТ

   Одна строка:

       DT_REP
       CLI_ID
       CUR

       DEPOSIT_BALANCE
       NS_BALANCE

       DEPOSIT_RATE
       NS_RATE

       EARLY_CLOSE_FLAG
       EARLY_CLOSE_COUNT
       EARLY_CLOSE_RATE
       EARLY_CLOSE_BALANCE_RUB
   ============================================================ */

SELECT
      c.dt_rep
    , cc.cli_id
    , cc.cur


    /* Остаток вкладов */
    , ISNULL(
          d.deposit_balance,
          CAST(0 AS decimal(38,6))
      ) AS deposit_balance


    /* Остаток НС */
    , ISNULL(
          d.ns_balance,
          CAST(0 AS decimal(38,6))
      ) AS ns_balance


    /* Средневзвешенная ставка вкладов */
    , d.deposit_rate


    /* Средневзвешенная ставка НС */
    , d.ns_rate


    /* Был ли досрок */
    , ISNULL(
          e.early_close_flag,
          0
      ) AS early_close_flag


    /* Количество досрочно закрытых вкладов */
    , ISNULL(
          e.early_close_count,
          0
      ) AS early_close_count


    /* Средневзвешенная ставка досрочно закрытых вкладов */
    , e.early_close_rate


    /* Рублёвый объём досрочно закрытых вкладов */
    , ISNULL(
          e.early_close_balance_rub,
          CAST(0 AS decimal(38,6))
      ) AS early_close_balance_rub


FROM #calendar c

CROSS JOIN #client_currency cc


LEFT JOIN #daily_agg d
    ON  d.dt_rep = c.dt_rep
    AND d.cli_id = cc.cli_id
    AND d.cur    = cc.cur


LEFT JOIN #early_close e
    ON  e.dt_rep = c.dt_rep
    AND e.cli_id = cc.cli_id
    AND e.cur    = cc.cur


ORDER BY
      c.dt_rep
    , cc.cli_id
    , cc.cur

OPTION (RECOMPILE);



/* ============================================================
   15. ОТДЕЛЬНЫЙ RESULT SET:
       ВСЕ ЖИВЫЕ ВКЛАДЫ НА @DateTo

   Живой на дату:

       DT_OPEN <= @DateTo

       AND

       DT_CLOSE IS NULL
       OR DT_CLOSE >= @DateTo

   Возвращаем ВСЕ ПОЛЯ исходного снапшота.

   Для каждого CON_ID берём последнюю запись
   не позднее @DateTo.
   ============================================================ */

SELECT
    src.*

FROM #deposit_contracts dc

CROSS APPLY
(
    SELECT TOP (1)
        d.*

    FROM [ALM_TEST].[WORK].[DepositInterestsRateSnap] d WITH (NOLOCK)

    WHERE
        d.CON_ID = dc.con_id

        AND CAST(d.DT_REP AS date) <= @DateTo

    ORDER BY
        d.DT_REP DESC

) src

WHERE
    dc.dt_open <= @DateTo

    AND
    (
        dc.dt_close IS NULL
        OR dc.dt_close >= @DateTo
    )

ORDER BY
      dc.cli_id
    , dc.con_id;



/* ============================================================
   16. ОТДЕЛЬНЫЙ RESULT SET:
       ВСЕ ЖИВЫЕ НАКОПИТЕЛЬНЫЕ СЧЕТА НА @DateTo

   Только два продукта:

       Накопительный счёт
       Накопительный счёт Ультра

   Возвращаем ВСЕ ПОЛЯ исходной таблицы.
   ============================================================ */

SELECT
    src.*

FROM #ns_contracts nc

CROSS APPLY
(
    SELECT TOP (1)
        n.*

    FROM [LIQUIDITY].[liq].[depositcontract_all] n WITH (NOLOCK)

    WHERE
        n.CON_ID = nc.con_id

        AND n.CLI_SHORT_NAME = N'ФЛ'

        AND n.PROD_NAME IN
        (
              N'Накопительный счёт Ультра'
            , N'Накопительный счёт'
        )

    ORDER BY
          n.DT_UPDATE DESC
        , n.DT_IMPORT DESC

) src

WHERE
    nc.dt_open <= @DateTo

    AND
    (
        nc.dt_close IS NULL
        OR nc.dt_close >= @DateTo
    )

ORDER BY
      nc.cli_id
    , nc.con_id;
