USE [ALM_TEST];

SET NOCOUNT ON;


/* ================================================================
   0. ПАРАМЕТРЫ
   ================================================================ */

DECLARE @DateFrom date = '2026-01-01';
DECLARE @DateTo   date = '2026-09-26';

/* Нужен только для определения исчезновений 01.01.2026 */
DECLARE @SeedDate date = DATEADD(day, -1, @DateFrom);



/* ================================================================
   1. КЛИЕНТЫ

   МЕНЯТЬ ТОЛЬКО ЭТИ ТРИ CLI_ID
   ================================================================ */

DROP TABLE IF EXISTS #clients;

CREATE TABLE #clients
(
    cli_id bigint NOT NULL
);

INSERT INTO #clients (cli_id)
VALUES
      (1111111111)
    , (2222222222)
    , (3333333333);



/* ================================================================
   2. ДНЕВНАЯ СЫРАЯ ВЫГРУЗКА

   Создаём структуру из самого VIEW, но без строк.

   В #day_balance в каждый момент времени находится
   ТОЛЬКО ОДИН ДЕНЬ.

   На последней дате 26.09.2026 таблицу не очищаем,
   чтобы потом вывести все исходные поля живых договоров.
   ================================================================ */

DROP TABLE IF EXISTS #day_balance;

SELECT TOP (0)
    b.*
INTO #day_balance
FROM [ALM].[ALM].[vw_balance_rest_all] b;



/* ================================================================
   3. КОМПАКТНЫЙ СПИСОК ДОГОВОРОВ ТЕКУЩЕГО ДНЯ

   Одна строка:
       DATE + CLI_ID + CON_ID + SECTION + CUR

   Даже если в исходном балансе по одному CON_ID несколько строк,
   здесь они схлопываются.
   ================================================================ */

DROP TABLE IF EXISTS #curr_contracts;

CREATE TABLE #curr_contracts
(
      dt_rep        date
    , cli_id        bigint
    , con_id        bigint
    , section_name  nvarchar(100)
    , cur           nvarchar(20)

    , out_rub       decimal(38,6)
    , rate_con      decimal(18,10)

    , dt_open       date
    , dt_close      date
);



/* ================================================================
   4. ДОГОВОРЫ ПРЕДЫДУЩЕГО ДНЯ

   Нужны только для определения:

       был вчера
       +
       сегодня отсутствует

       => считаем досрочным исчезновением
   ================================================================ */

DROP TABLE IF EXISTS #prev_contracts;

CREATE TABLE #prev_contracts
(
      dt_rep        date
    , cli_id        bigint
    , con_id        bigint
    , section_name  nvarchar(100)
    , cur           nvarchar(20)

    , out_rub       decimal(38,6)
    , rate_con      decimal(18,10)

    , dt_open       date
    , dt_close      date
);



/* ================================================================
   5. КОМПАКТНЫЕ ДНЕВНЫЕ РЕЗУЛЬТАТЫ

   Здесь уже всего несколько строк на день.
   ================================================================ */

DROP TABLE IF EXISTS #daily_result;

CREATE TABLE #daily_result
(
      dt_rep              date
    , cli_id              bigint
    , cur                 nvarchar(20)

    , deposit_balance     decimal(38,6)
    , ns_balance          decimal(38,6)

    , deposit_rate        decimal(18,10)
    , ns_rate             decimal(18,10)
);



/* ================================================================
   6. ДОСРОЧНЫЕ ИСЧЕЗНОВЕНИЯ

   Здесь сразу агрегированные результаты,
   сырые закрытые договоры хранить не нужно.
   ================================================================ */

DROP TABLE IF EXISTS #early_close;

CREATE TABLE #early_close
(
      dt_rep                    date
    , cli_id                    bigint
    , cur                       nvarchar(20)

    , early_close_flag          tinyint
    , early_close_count         int

    , early_close_rate          decimal(18,10)
    , early_close_balance_rub   decimal(38,6)
);



/* ================================================================
   7. СПИСОК ВАЛЮТ КЛИЕНТОВ

   Маленькая таблица.
   Нужна, чтобы в финале построить полное ежедневное полотно,
   включая дни с нулевым остатком.
   ================================================================ */

DROP TABLE IF EXISTS #client_currency;

CREATE TABLE #client_currency
(
      cli_id bigint
    , cur    nvarchar(20)
);



/* ================================================================
   8. ЗАГРУЖАЕМ ТОЛЬКО 31.12.2025

   Это стартовый снимок для сравнения с 01.01.2026.

   К VIEW ОБРАЩАЕМСЯ РОВНО ОДИН РАЗ.
   ================================================================ */

TRUNCATE TABLE #day_balance;


INSERT INTO #day_balance

SELECT
    b.*

FROM [ALM].[ALM].[vw_balance_rest_all] b WITH (NOLOCK)

INNER JOIN #clients c
    ON c.cli_id = b.cli_id

WHERE
    b.dt_rep >= @SeedDate

    AND b.dt_rep < DATEADD(day, 1, @SeedDate)

    AND b.block_name = N'Привлечение ФЛ'

    AND b.section_name IN
    (
          N'Срочные'
        , N'Накопительный счет'
    )

    AND b.od_flag = 1

    AND b.con_id IS NOT NULL

OPTION (MAXDOP 1);



/* ================================================================
   9. ИЗ 31.12 СОЗДАЁМ PREV_CONTRACTS

   После этого сырой баланс 31.12 больше не нужен.
   ================================================================ */

INSERT INTO #prev_contracts
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
      @SeedDate

    , CAST(b.cli_id AS bigint)
    , CAST(b.con_id AS bigint)

    , CAST(b.section_name AS nvarchar(100))


    /* Нормализация валюты */
    , CASE
          WHEN b.cur IS NULL
              THEN N'UNKNOWN'

          WHEN UPPER(
                   LTRIM(
                       RTRIM(
                           CAST(b.cur AS nvarchar(20))
                       )
                   )
               ) IN
               (
                   N'810',
                   N'643',
                   N'RUR',
                   N'RUB'
               )
              THEN N'RUR'

          ELSE
              UPPER(
                  LTRIM(
                      RTRIM(
                          CAST(b.cur AS nvarchar(20))
                      )
                  )
              )
      END AS cur


    /* Остаток договора */
    , SUM(
          ISNULL(
              TRY_CAST(
                  b.out_rub AS decimal(38,6)
              ),
              0
          )
      ) AS out_rub


    /* Ставка договора.
       В нормальной ситуации по одному CON_ID одна ставка. */
    , MAX(
          TRY_CAST(
              b.rate_con AS decimal(18,10)
          )
      ) AS rate_con


    , MIN(
          TRY_CAST(
              b.dt_open AS date
          )
      ) AS dt_open


    , MAX(
          TRY_CAST(
              b.dt_close AS date
          )
      ) AS dt_close


FROM #day_balance b

GROUP BY
      b.cli_id
    , b.con_id
    , b.section_name

    , CASE
          WHEN b.cur IS NULL
              THEN N'UNKNOWN'

          WHEN UPPER(
                   LTRIM(
                       RTRIM(
                           CAST(b.cur AS nvarchar(20))
                       )
                   )
               ) IN
               (
                   N'810',
                   N'643',
                   N'RUR',
                   N'RUB'
               )
              THEN N'RUR'

          ELSE
              UPPER(
                  LTRIM(
                      RTRIM(
                          CAST(b.cur AS nvarchar(20))
                      )
                  )
              )
      END;



/* Валюты также запоминаем */

INSERT INTO #client_currency
(
      cli_id
    , cur
)

SELECT DISTINCT
      p.cli_id
    , p.cur

FROM #prev_contracts p;



/* ================================================================
   10. ОСНОВНОЙ ЦИКЛ

   01.01.2026 - 26.09.2026

   ВАЖНО:

   на каждой итерации:

   1. очищаем сырые данные прошлого дня
   2. ОДИН РАЗ вызываем vw_balance_rest_all на новую дату
   3. сразу схлопываем данные до договоров
   4. считаем агрегаты
   5. ищем исчезнувшие вклады
   6. текущие договоры становятся предыдущими
   7. идём дальше

   Поэтому сырые данные 269 дней одновременно
   НИКОГДА не хранятся.
   ================================================================ */

DECLARE @CurrentDate date = @DateFrom;


WHILE @CurrentDate <= @DateTo
BEGIN


    /* ============================================================
       10.1. УБИРАЕМ СЫРЬЁ ПРЕДЫДУЩЕГО ДНЯ
       ============================================================ */

    TRUNCATE TABLE #day_balance;

    TRUNCATE TABLE #curr_contracts;



    /* ============================================================
       10.2. РОВНО ОДИН ЗАПРОС К БАЛАНСУ НА ЭТУ ДАТУ
       ============================================================ */

    INSERT INTO #day_balance

    SELECT
        b.*

    FROM [ALM].[ALM].[vw_balance_rest_all] b WITH (NOLOCK)

    INNER JOIN #clients c
        ON c.cli_id = b.cli_id

    WHERE
        b.dt_rep >= @CurrentDate

        AND b.dt_rep < DATEADD(day, 1, @CurrentDate)


        AND b.block_name = N'Привлечение ФЛ'


        AND b.section_name IN
        (
              N'Срочные'
            , N'Накопительный счет'
        )


        AND b.od_flag = 1


        AND b.con_id IS NOT NULL


    /* Не даём небольшому пользовательскому запросу
       занимать много параллельных потоков сервера */
    OPTION (MAXDOP 1);



    /* ============================================================
       10.3. СХЛОПЫВАЕМ СЫРОЙ ДЕНЬ ДО УРОВНЯ ДОГОВОРА
       ============================================================ */

    INSERT INTO #curr_contracts
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
          @CurrentDate

        , CAST(b.cli_id AS bigint)

        , CAST(b.con_id AS bigint)

        , CAST(
              b.section_name AS nvarchar(100)
          )


        /* Валюта */
        , CASE
              WHEN b.cur IS NULL
                  THEN N'UNKNOWN'

              WHEN UPPER(
                       LTRIM(
                           RTRIM(
                               CAST(b.cur AS nvarchar(20))
                           )
                       )
                   ) IN
                   (
                       N'810',
                       N'643',
                       N'RUR',
                       N'RUB'
                   )
                  THEN N'RUR'

              ELSE
                  UPPER(
                      LTRIM(
                          RTRIM(
                              CAST(b.cur AS nvarchar(20))
                          )
                      )
                  )
          END AS cur


        /* Остаток договора */
        , SUM(
              ISNULL(
                  TRY_CAST(
                      b.out_rub AS decimal(38,6)
                  ),
                  0
              )
          ) AS out_rub


        /* Ставка */
        , MAX(
              TRY_CAST(
                  b.rate_con AS decimal(18,10)
              )
          ) AS rate_con


        , MIN(
              TRY_CAST(
                  b.dt_open AS date
              )
          ) AS dt_open


        , MAX(
              TRY_CAST(
                  b.dt_close AS date
              )
          ) AS dt_close


    FROM #day_balance b


    GROUP BY
          b.cli_id
        , b.con_id
        , b.section_name

        , CASE
              WHEN b.cur IS NULL
                  THEN N'UNKNOWN'

              WHEN UPPER(
                       LTRIM(
                           RTRIM(
                               CAST(b.cur AS nvarchar(20))
                           )
                       )
                   ) IN
                   (
                       N'810',
                       N'643',
                       N'RUR',
                       N'RUB'
                   )
                  THEN N'RUR'

              ELSE
                  UPPER(
                      LTRIM(
                          RTRIM(
                              CAST(b.cur AS nvarchar(20))
                          )
                      )
                  )
          END;



    /* ============================================================
       10.4. ЗАПОМИНАЕМ НОВЫЕ ВАЛЮТЫ

       Таблица микроскопическая, поэтому индекс тут не нужен.
       ============================================================ */

    INSERT INTO #client_currency
    (
          cli_id
        , cur
    )

    SELECT DISTINCT
          d.cli_id
        , d.cur

    FROM #curr_contracts d

    WHERE NOT EXISTS
    (
        SELECT 1

        FROM #client_currency cc

        WHERE
            cc.cli_id = d.cli_id
            AND cc.cur = d.cur
    );



    /* ============================================================
       10.5. ДНЕВНЫЕ ОСТАТКИ И СТАВКИ
       ============================================================ */

    INSERT INTO #daily_result
    (
          dt_rep
        , cli_id
        , cur

        , deposit_balance
        , ns_balance

        , deposit_rate
        , ns_rate
    )

    SELECT
          @CurrentDate

        , d.cli_id
        , d.cur


        /* --------------------------------------------------------
           Срочные вклады
           -------------------------------------------------------- */
        , SUM(
              CASE
                  WHEN d.section_name = N'Срочные'
                      THEN d.out_rub

                  ELSE 0
              END
          ) AS deposit_balance


        /* --------------------------------------------------------
           Накопительные счета
           -------------------------------------------------------- */
        , SUM(
              CASE
                  WHEN d.section_name = N'Накопительный счет'
                      THEN d.out_rub

                  ELSE 0
              END
          ) AS ns_balance


        /* --------------------------------------------------------
           Средневзвешенная ставка срочных вкладов
           -------------------------------------------------------- */
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


        /* --------------------------------------------------------
           Средневзвешенная ставка накопительных счетов
           -------------------------------------------------------- */
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


    FROM #curr_contracts d

    GROUP BY
          d.cli_id
        , d.cur;



    /* ============================================================
       10.6. ДОСРОЧНЫЕ ИСЧЕЗНОВЕНИЯ

       Берём только вчерашние СРОЧНЫЕ вклады.

       Если вчера CON_ID был,
       а сегодня CON_ID отсутствует вообще,
       считаем его закрытым.

       Остаток и ставка =
       данные последнего дня существования договора.

       НС здесь НЕ считаются досрочными закрытиями.
       ============================================================ */

    INSERT INTO #early_close
    (
          dt_rep
        , cli_id
        , cur

        , early_close_flag
        , early_close_count

        , early_close_rate
        , early_close_balance_rub
    )

    SELECT
          @CurrentDate

        , p.cli_id
        , p.cur

        , CAST(1 AS tinyint)


        /* Сколько договоров исчезло */
        , COUNT(*) AS early_close_count


        /* Средневзвешенная ставка исчезнувших договоров */
        , CAST(

              SUM(
                  CASE
                      WHEN p.rate_con IS NOT NULL

                          THEN
                              p.out_rub
                              * p.rate_con

                      ELSE 0
                  END
              )

              /

              NULLIF(
                  SUM(
                      CASE
                          WHEN p.rate_con IS NOT NULL
                              THEN p.out_rub

                          ELSE 0
                      END
                  ),
                  0
              )

          AS decimal(18,10)) AS early_close_rate


        /* Остаток на последний день жизни */
        , SUM(
              p.out_rub
          ) AS early_close_balance_rub


    FROM #prev_contracts p


    WHERE
        p.section_name = N'Срочные'


        /* Сегодня CON_ID отсутствует */
        AND NOT EXISTS
        (
            SELECT 1

            FROM #curr_contracts n

            WHERE
                n.cli_id = p.cli_id

                AND n.con_id = p.con_id
        )


    GROUP BY
          p.cli_id
        , p.cur;



    /* ============================================================
       10.7. ТЕКУЩИЙ ДЕНЬ СТАНОВИТСЯ ПРЕДЫДУЩИМ

       Старый предыдущий день полностью удаляется.
       ============================================================ */

    TRUNCATE TABLE #prev_contracts;


    INSERT INTO #prev_contracts
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
          dt_rep
        , cli_id
        , con_id
        , section_name
        , cur
        , out_rub
        , rate_con
        , dt_open
        , dt_close

    FROM #curr_contracts;



    /* ============================================================
       10.8. СЛЕДУЮЩАЯ ДАТА
       ============================================================ */

    SET @CurrentDate =
        DATEADD(day, 1, @CurrentDate);

END;



/* ================================================================
   11. RESULT SET №1
       ПОЛНОЕ ПОДНЕВНОЕ ПОЛОТНО

   После завершения цикла тяжёлый VIEW больше не вызывается.
   ================================================================ */

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
      cal.dt_rep

    , cc.cli_id

    , cc.cur


    /* ------------------------------------------------------------
       Остатки
       ------------------------------------------------------------ */

    , ISNULL(
          d.deposit_balance,
          CAST(0 AS decimal(38,6))
      ) AS deposit_balance


    , ISNULL(
          d.ns_balance,
          CAST(0 AS decimal(38,6))
      ) AS ns_balance


    /* ------------------------------------------------------------
       Ставки
       ------------------------------------------------------------ */

    , d.deposit_rate

    , d.ns_rate


    /* ------------------------------------------------------------
       Досрочные исчезновения
       ------------------------------------------------------------ */

    , ISNULL(
          e.early_close_flag,
          0
      ) AS early_close_flag


    , ISNULL(
          e.early_close_count,
          0
      ) AS early_close_count


    , e.early_close_rate


    , ISNULL(
          e.early_close_balance_rub,
          CAST(0 AS decimal(38,6))
      ) AS early_close_balance_rub


FROM calendar cal

CROSS JOIN #client_currency cc


LEFT JOIN #daily_result d
    ON  d.dt_rep = cal.dt_rep
    AND d.cli_id = cc.cli_id
    AND d.cur = cc.cur


LEFT JOIN #early_close e
    ON  e.dt_rep = cal.dt_rep
    AND e.cli_id = cc.cli_id
    AND e.cur = cc.cur


ORDER BY
      cal.dt_rep
    , cc.cli_id
    , cc.cur

OPTION (MAXRECURSION 0);



/* ================================================================
   12. RESULT SET №2
       ВСЕ ЖИВЫЕ ВКЛАДЫ + НС НА 26.09.2026

   НИКАКОГО ПОВТОРНОГО ОБРАЩЕНИЯ К VIEW НЕТ.

   В #day_balance после окончания цикла осталась
   именно выгрузка @DateTo = 26.09.2026.

   Выводим ВСЕ исходные поля VIEW.
   ================================================================ */

SELECT
      CASE
          WHEN b.section_name = N'Срочные'
              THEN N'DEPOSIT'

          WHEN b.section_name = N'Накопительный счет'
              THEN N'NS'

          ELSE N'OTHER'
      END AS balance_type

    , b.*

FROM #day_balance b

ORDER BY
      b.cli_id
    , b.section_name
    , b.cur
    , b.con_id;
