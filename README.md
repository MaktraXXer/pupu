/* ================================================================
   ПОДНЕВНАЯ ДИНАМИКА ВКЛАДОВ И НС ПО 3 КЛИЕНТАМ
   ИСТОЧНИК: ALM.ALM.vw_balance_rest_all

   ПЕРИОД:
       01.01.2026 - 26.09.2026

   ФИЛЬТРЫ:
       block_name   = 'Привлечение ФЛ'
       section_name IN ('Срочные', 'Накопительный счет')
       od_flag      = 1

   РЕЗУЛЬТАТ 1:
       dt_rep
       cli_id
       cur
       deposit_balance
       ns_balance
       deposit_rate
       ns_rate
       early_close_flag
       early_close_count
       early_close_rate
       early_close_balance_rub

   РЕЗУЛЬТАТ 2:
       все строки живых вкладов + НС на 26.09.2026
       со всеми полями исходного vw_balance_rest_all
   ================================================================ */

SET NOCOUNT ON;


/* ================================================================
   0. ПАРАМЕТРЫ
   ================================================================ */

DECLARE @DateFrom date = '2026-01-01';
DECLARE @DateTo   date = '2026-09-26';



/* ================================================================
   1. ТРИ КЛИЕНТА

   МЕНЯТЬ ТОЛЬКО ЭТИ ЗНАЧЕНИЯ
   ================================================================ */

DROP TABLE IF EXISTS #clients;

CREATE TABLE #clients
(
    cli_id bigint NOT NULL PRIMARY KEY
);

INSERT INTO #clients (cli_id)
VALUES
      (1111111111)      -- CLI_ID 1
    , (2222222222)      -- CLI_ID 2
    , (3333333333);     -- CLI_ID 3



/* ================================================================
   2. ТАБЛИЦА ДЛЯ ПОДНЕВНЫХ СНИМКОВ

   Сюда циклом складываем только нужные строки.

   ВАЖНО:
   никакой DepositContract_Saldo здесь больше нет.
   ================================================================ */

DROP TABLE IF EXISTS #balance_raw;

CREATE TABLE #balance_raw
(
      dt_rep       date           NOT NULL
    , cli_id       bigint         NOT NULL
    , con_id       bigint         NOT NULL

    , section_name nvarchar(100)  NOT NULL
    , cur           varchar(20)    NOT NULL

    , out_rub       decimal(38,6)  NULL
    , rate_con      decimal(18,10) NULL

    , dt_open       date           NULL
    , dt_close      date           NULL
);



/* ================================================================
   3. ЦИКЛ ПО ДАТАМ

   На каждой дате обращаемся к vw_balance_rest_all
   только за:
       - 3 клиентами
       - Привлечением ФЛ
       - Срочными + НС
       - od_flag = 1

   Это специально сделано циклом, чтобы не вытаскивать
   огромный диапазон баланса целиком.
   ================================================================ */

DECLARE @CurrentDate date = @DateFrom;

WHILE @CurrentDate <= @DateTo
BEGIN

    INSERT INTO #balance_raw
    (
          dt_rep
        , cli_id
        , con_id
        , section_name
        , cur
        , out_rub
        , rate_con
        , dt_open
        , dt_close
    )

    SELECT
          CAST(b.dt_rep AS date)

        , CAST(b.cli_id AS bigint)

        , CAST(b.con_id AS bigint)

        , CAST(b.section_name AS nvarchar(100))


        /* --------------------------------------------------------
           Нормализуем рубли:

           810 / 643 / RUB / RUR -> RUR
           -------------------------------------------------------- */
        , CASE
              WHEN UPPER(
                       LTRIM(
                           RTRIM(
                               CAST(b.cur AS varchar(20))
                           )
                       )
                   ) IN ('810', '643', 'RUB', 'RUR')
                  THEN 'RUR'

              WHEN b.cur IS NULL
                  THEN 'UNKNOWN'

              ELSE
                  UPPER(
                      LTRIM(
                          RTRIM(
                              CAST(b.cur AS varchar(20))
                          )
                      )
                  )
          END AS cur


        /* Остаток */
        , TRY_CAST(
              b.out_rub AS decimal(38,6)
          ) AS out_rub


        /* Ставка договора */
        , TRY_CAST(
              b.rate_con AS decimal(18,10)
          ) AS rate_con


        , CAST(b.dt_open AS date)

        , CAST(b.dt_close AS date)


    FROM [ALM].[ALM].[vw_balance_rest_all] b WITH (NOLOCK)

    INNER JOIN #clients c
        ON c.cli_id = b.cli_id


    WHERE

        /* Используем диапазон, чтобы не CAST'овать поле в WHERE */
        b.dt_rep >= @CurrentDate

        AND b.dt_rep < DATEADD(day, 1, @CurrentDate)


        AND b.block_name = N'Привлечение ФЛ'


        AND b.section_name IN
        (
              N'Срочные'
            , N'Накопительный счет'
        )


        AND b.od_flag = 1


    OPTION (RECOMPILE);


    SET @CurrentDate =
        DATEADD(day, 1, @CurrentDate);

END;



/* Индекс ставим уже после загрузки */

CREATE CLUSTERED INDEX IX_balance_raw
ON #balance_raw
(
      dt_rep
    , cli_id
    , con_id
);



/* ================================================================
   4. СХЛОПЫВАЕМ ДУБЛИ ОДНОГО ДОГОВОРА

   Если один CON_ID вдруг представлен несколькими строками
   внутри одного DT_REP, сначала агрегируем его.

   Одна строка:

       DT_REP
       CLI_ID
       CON_ID
       SECTION_NAME
       CUR
   ================================================================ */

DROP TABLE IF EXISTS #contract_daily;

SELECT
      dt_rep
    , cli_id
    , con_id
    , section_name
    , cur


    /* Остаток договора */
    , SUM(
          ISNULL(out_rub, 0)
      ) AS out_rub


    /* У одного договора ставка должна быть одна.
       MAX защищает от технических дублей. */
    , MAX(rate_con) AS rate_con


    , MIN(dt_open) AS dt_open

    , MAX(dt_close) AS dt_close


INTO #contract_daily

FROM #balance_raw

GROUP BY
      dt_rep
    , cli_id
    , con_id
    , section_name
    , cur;


CREATE UNIQUE CLUSTERED INDEX IX_contract_daily
ON #contract_daily
(
      dt_rep
    , cli_id
    , con_id
    , section_name
    , cur
);



CREATE NONCLUSTERED INDEX IX_contract_daily_con
ON #contract_daily
(
      cli_id
    , con_id
    , dt_rep
)

INCLUDE
(
      section_name
    , cur
    , out_rub
    , rate_con
);



/* ================================================================
   5. КАЛЕНДАРЬ
   ================================================================ */

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



/* ================================================================
   6. СПИСОК ВАЛЮТ КАЖДОГО КЛИЕНТА

   Благодаря этому получим полное полотно:
       дата × клиент × валюта

   даже когда на конкретную дату остаток = 0.
   ================================================================ */

DROP TABLE IF EXISTS #client_currency;

SELECT DISTINCT
      cli_id
    , cur

INTO #client_currency

FROM #contract_daily;


CREATE UNIQUE CLUSTERED INDEX IX_client_currency
ON #client_currency
(
      cli_id
    , cur
);



/* ================================================================
   7. ОСНОВНЫЕ ПОДНЕВНЫЕ ПОКАЗАТЕЛИ

   Срочные:
       deposit_balance
       deposit_rate

   Накопительный счет:
       ns_balance
       ns_rate

   Ставка:
       SUM(out_rub * rate_con)
       -----------------------
           SUM(out_rub)
   ================================================================ */

DROP TABLE IF EXISTS #daily_agg;

SELECT
      d.dt_rep
    , d.cli_id
    , d.cur


    /* ============================================================
       ОСТАТОК СРОЧНЫХ ВКЛАДОВ
       ============================================================ */

    , SUM(
          CASE
              WHEN d.section_name = N'Срочные'
                  THEN d.out_rub

              ELSE 0
          END
      ) AS deposit_balance


    /* ============================================================
       ОСТАТОК НАКОПИТЕЛЬНЫХ СЧЕТОВ
       ============================================================ */

    , SUM(
          CASE
              WHEN d.section_name = N'Накопительный счет'
                  THEN d.out_rub

              ELSE 0
          END
      ) AS ns_balance


    /* ============================================================
       СРЕДНЕВЗВЕШЕННАЯ СТАВКА ВКЛАДОВ
       ============================================================ */

    , CAST(

          SUM(
              CASE
                  WHEN d.section_name = N'Срочные'
                       AND d.rate_con IS NOT NULL

                      THEN
                          d.out_rub
                          * d.rate_con

                  ELSE 0
              END
          )

          /

          NULLIF(
              SUM(
                  CASE
                      WHEN d.section_name = N'Срочные'
                           AND d.rate_con IS NOT NULL

                          THEN d.out_rub

                      ELSE 0
                  END
              ),
              0
          )

      AS decimal(18,10)) AS deposit_rate


    /* ============================================================
       СРЕДНЕВЗВЕШЕННАЯ СТАВКА НС
       ============================================================ */

    , CAST(

          SUM(
              CASE
                  WHEN d.section_name = N'Накопительный счет'
                       AND d.rate_con IS NOT NULL

                      THEN
                          d.out_rub
                          * d.rate_con

                  ELSE 0
              END
          )

          /

          NULLIF(
              SUM(
                  CASE
                      WHEN d.section_name = N'Накопительный счет'
                           AND d.rate_con IS NOT NULL

                          THEN d.out_rub

                      ELSE 0
                  END
              ),
              0
          )

      AS decimal(18,10)) AS ns_rate


INTO #daily_agg

FROM #contract_daily d

GROUP BY
      d.dt_rep
    , d.cli_id
    , d.cur;


CREATE UNIQUE CLUSTERED INDEX IX_daily_agg
ON #daily_agg
(
      dt_rep
    , cli_id
    , cur
);



/* ================================================================
   8. ОПРЕДЕЛЯЕМ ДОСРОЧНО ИСЧЕЗНУВШИЕ ВКЛАДЫ

   ЛОГИКА:

   если срочный вклад:

       есть в балансе на дату T

   но:

       отсутствует в балансе на T + 1

   значит на T + 1 считаем событие досрочного закрытия.


   Например:

       10.05:
           CON_ID 123
           остаток = 1 000 000
           RATE = 15%

       11.05:
           CON_ID 123 отсутствует

   Тогда на 11.05:

       early_close_flag        = 1
       early_close_count       = 1
       early_close_balance_rub = 1 000 000
       early_close_rate        = 15%

   ВАЖНО:
   объём и ставка берутся из последнего дня,
   когда вклад ещё был жив.
   ================================================================ */

DROP TABLE IF EXISTS #early_close_detail;

SELECT
      DATEADD(day, 1, p.dt_rep) AS dt_rep

    , p.cli_id
    , p.cur
    , p.con_id

    , p.out_rub AS balance_before_close

    , p.rate_con AS rate_before_close

    , p.dt_open
    , p.dt_close


INTO #early_close_detail

FROM #contract_daily p


WHERE
    p.section_name = N'Срочные'


    /* Следующая дата должна входить в горизонт */
    AND p.dt_rep < @DateTo


    /* На следующий календарный день
       этого CON_ID уже нет */
    AND NOT EXISTS
    (
        SELECT 1

        FROM #contract_daily n

        WHERE
            n.cli_id = p.cli_id

            AND n.con_id = p.con_id

            AND n.dt_rep =
                DATEADD(day, 1, p.dt_rep)
    );


CREATE UNIQUE CLUSTERED INDEX IX_early_close_detail
ON #early_close_detail
(
      dt_rep
    , cli_id
    , cur
    , con_id
);



/* ================================================================
   9. АГРЕГАЦИЯ ДОСРОЧЕК

   На дату + клиента + валюту:

   - флаг
   - число закрывшихся вкладов
   - рублёвый объём
   - средневзвешенная ставка
   ================================================================ */

DROP TABLE IF EXISTS #early_close;

SELECT
      dt_rep
    , cli_id
    , cur


    /* Было ли исчезновение хотя бы одного вклада */
    , CAST(1 AS tinyint) AS early_close_flag


    /* Сколько договоров исчезло */
    , COUNT(*) AS early_close_count


    /* Сколько денег было на них
       в последний день перед исчезновением */
    , SUM(
          balance_before_close
      ) AS early_close_balance_rub


    /* Средневзвешенная ставка исчезнувших вкладов */
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
      dt_rep
    , cli_id
    , cur;


CREATE UNIQUE CLUSTERED INDEX IX_early_close
ON #early_close
(
      dt_rep
    , cli_id
    , cur
);



/* ================================================================
   10. РЕЗУЛЬТАТ №1
       ПОЛНОЕ ПОДНЕВНОЕ ПОЛОТНО

   Одна строка:
       DT_REP + CLI_ID + CUR
   ================================================================ */

SELECT
      cal.dt_rep

    , cc.cli_id

    , cc.cur


    /* ============================================================
       ОСТАТКИ
       ============================================================ */

    , ISNULL(
          d.deposit_balance,
          CAST(0 AS decimal(38,6))
      ) AS deposit_balance


    , ISNULL(
          d.ns_balance,
          CAST(0 AS decimal(38,6))
      ) AS ns_balance


    /* ============================================================
       СТАВКИ
       ============================================================ */

    , d.deposit_rate

    , d.ns_rate


    /* ============================================================
       ДОСРОЧКА
       ============================================================ */

    , ISNULL(
          e.early_close_flag,
          0
      ) AS early_close_flag


    /* Количество вкладов,
       исчезнувших из баланса в этот день */
    , ISNULL(
          e.early_close_count,
          0
      ) AS early_close_count


    /* Средневзвешенная ставка
       этих вкладов в последний день до исчезновения */
    , e.early_close_rate


    /* Их суммарный остаток
       в последний день до исчезновения */
    , ISNULL(
          e.early_close_balance_rub,
          CAST(0 AS decimal(38,6))
      ) AS early_close_balance_rub


FROM #calendar cal

CROSS JOIN #client_currency cc


LEFT JOIN #daily_agg d
    ON  d.dt_rep = cal.dt_rep
    AND d.cli_id = cc.cli_id
    AND d.cur    = cc.cur


LEFT JOIN #early_close e
    ON  e.dt_rep = cal.dt_rep
    AND e.cli_id = cc.cli_id
    AND e.cur    = cc.cur


ORDER BY
      cal.dt_rep
    , cc.cli_id
    , cc.cur;



/* ================================================================
   11. РЕЗУЛЬТАТ №2
       ВСЕ ЖИВЫЕ ВКЛАДЫ И НАКОПИТЕЛЬНЫЕ СЧЕТА
       НА ПОСЛЕДНЮЮ ДАТУ

   ЕДИНЫЙ SELECT.

   Критерий "живой":
       строка реально присутствует
       в vw_balance_rest_all на @DateTo

   Возвращаем ВСЕ исходные поля.

   Дополнительно balance_type:
       DEPOSIT
       NS
   ================================================================ */

SELECT
      CASE
          WHEN b.section_name = N'Срочные'
              THEN 'DEPOSIT'

          WHEN b.section_name = N'Накопительный счет'
              THEN 'NS'

          ELSE 'OTHER'
      END AS balance_type

    , b.*


FROM [ALM].[ALM].[vw_balance_rest_all] b WITH (NOLOCK)

INNER JOIN #clients c
    ON c.cli_id = b.cli_id


WHERE
    b.dt_rep >= @DateTo

    AND b.dt_rep < DATEADD(day, 1, @DateTo)


    AND b.block_name = N'Привлечение ФЛ'


    AND b.section_name IN
    (
          N'Срочные'
        , N'Накопительный счет'
    )


    AND b.od_flag = 1


ORDER BY
      b.cli_id
    , b.section_name
    , b.cur
    , b.con_id

OPTION (RECOMPILE);
