
USE ALM;
SET NOCOUNT ON;

DECLARE @dt_from date = '2026-03-01';
DECLARE @dt_to   date = '2026-04-07';
DECLARE @dt_to_next date = DATEADD(day, 1, @dt_to);

DROP TABLE IF EXISTS #base;
DROP TABLE IF EXISTS #new_clients;
DROP TABLE IF EXISTS #contracts;
DROP TABLE IF EXISTS #daily_delta;
DROP TABLE IF EXISTS #daily;

/* ============================================================
   1. ИСТОРИЯ ДОГОВОРОВ ФЛ
   ============================================================ */

SELECT DISTINCT
      a.CLI_ID
    , a.CON_ID
    , a.PROD_NAME
    , CAST(a.DT_OPEN AS date) AS dt_open
INTO #base
FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
WHERE a.DT_OPEN IS NOT NULL
  AND a.DT_OPEN < @dt_to_next
  AND a.cli_short_name = N'ФЛ'
  AND a.cur = 'RUR'
  AND a.PROD_NAME NOT IN (
        N'Агентские эскроу ФЛ по ставке КС+спред',
        N'Спец. банк.счёт',
        N'Залоговый',
        N'Агентские эскроу ФЛ по ставке КС/2',
        N'Эскроу',
        N'Депозит ФЛ (суррогатный договор для счета Новой Афины)',
        N'Надёжный прайм',
        N'Надёжный VIP',
        N'Надёжный премиум',
        N'Надёжный промо',
        N'Надёжный старт',
        N'Надёжный Т2',
        N'Надёжный Мегафон',
        N'Надёжный процент',
        N'Могучий',
        N'Надёжный',
        N'ДОМа надёжно'
  );

CREATE INDEX IX_base_client
    ON #base (CLI_ID, dt_open);

/* ============================================================
   2. НОВЫЕ КЛИЕНТЫ РК

   Первый договор в доступной истории открыт в период РК.
   ============================================================ */

SELECT CLI_ID
INTO #new_clients
FROM #base
WHERE CLI_ID IS NOT NULL
GROUP BY CLI_ID
HAVING MIN(dt_open) >= @dt_from
   AND MIN(dt_open) <= @dt_to;

CREATE UNIQUE CLUSTERED INDEX IX_new_clients
    ON #new_clients (CLI_ID);

/* ============================================================
   3. ДОГОВОРЫ НОВЫХ КЛИЕНТОВ

   Только открытые в период РК.
   Один CON_ID = одна строка.
   ============================================================ */

;WITH ranked AS (
    SELECT
          b.CLI_ID
        , b.CON_ID
        , b.PROD_NAME
        , b.dt_open
        , ROW_NUMBER() OVER (
              PARTITION BY b.CON_ID
              ORDER BY b.dt_open, b.PROD_NAME
          ) AS rn
    FROM #base b
    INNER JOIN #new_clients n
        ON n.CLI_ID = b.CLI_ID
    WHERE b.dt_open BETWEEN @dt_from AND @dt_to
      AND b.CON_ID IS NOT NULL
)
SELECT
      CLI_ID
    , CON_ID
    , PROD_NAME
    , dt_open
    , CASE
          WHEN PROD_NAME LIKE N'Накопительный счёт%'
              THEN N'НС'
          ELSE N'Вклады'
      END AS product_type
INTO #contracts
FROM ranked
WHERE rn = 1;

CREATE UNIQUE CLUSTERED INDEX IX_contracts
    ON #contracts (CON_ID);

/* ============================================================
   4. ИСТОРИЯ САЛЬДО

   DT_FROM ... DT_TO — интервал действия OUT_RUB.
   Создаём +OUT_RUB в начале и -OUT_RUB после окончания.
   ============================================================ */

;WITH saldo_clipped AS (
    SELECT
          c.product_type
        , CAST(s.OUT_RUB AS decimal(38,6)) AS out_rub
        , bounds.dt_from
        , bounds.dt_to

    FROM LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)

    INNER JOIN #contracts c
        ON c.CON_ID = s.CON_ID

    CROSS APPLY (
        SELECT
            (
                SELECT MAX(v.dt)
                FROM (VALUES
                    (CAST(s.DT_FROM AS date)),
                    (c.dt_open),
                    (@dt_from)
                ) v(dt)
            ) AS dt_from,

            (
                SELECT MIN(v.dt)
                FROM (VALUES
                    (ISNULL(CAST(s.DT_TO AS date), @dt_to)),
                    (@dt_to)
                ) v(dt)
            ) AS dt_to
    ) bounds

    WHERE s.DT_FROM < @dt_to_next
      AND (s.DT_TO IS NULL OR s.DT_TO >= @dt_from)
      AND s.OUT_RUB IS NOT NULL
      AND bounds.dt_from <= bounds.dt_to
),

events AS (
    SELECT
          e.dt_rep
        , s.product_type
        , e.delta
    FROM saldo_clipped s
    CROSS APPLY (VALUES
          (s.dt_from, s.out_rub)
        , (
              CASE
                  WHEN s.dt_to < @dt_to
                  THEN DATEADD(day, 1, s.dt_to)
              END,
              -s.out_rub
          )
    ) e(dt_rep, delta)
    WHERE e.dt_rep IS NOT NULL
)

SELECT
      dt_rep
    , SUM(CASE WHEN product_type = N'Вклады'
               THEN delta ELSE 0 END) AS delta_deposits
    , SUM(CASE WHEN product_type = N'НС'
               THEN delta ELSE 0 END) AS delta_ns
INTO #daily_delta
FROM events
GROUP BY dt_rep;

/* ============================================================
   5. КАЛЕНДАРЬ И ПОДНЕВНЫЕ ОСТАТКИ
   ============================================================ */

;WITH calendar AS (
    SELECT @dt_from AS dt_rep

    UNION ALL

    SELECT DATEADD(day, 1, dt_rep)
    FROM calendar
    WHERE dt_rep < @dt_to
)
SELECT
      c.dt_rep

    , SUM(ISNULL(d.delta_deposits, 0)) OVER (
          ORDER BY c.dt_rep
          ROWS BETWEEN UNBOUNDED PRECEDING
                   AND CURRENT ROW
      ) AS balance_deposits

    , SUM(ISNULL(d.delta_ns, 0)) OVER (
          ORDER BY c.dt_rep
          ROWS BETWEEN UNBOUNDED PRECEDING
                   AND CURRENT ROW
      ) AS balance_ns

INTO #daily
FROM calendar c
LEFT JOIN #daily_delta d
    ON d.dt_rep = c.dt_rep
OPTION (MAXRECURSION 0);

/* ============================================================
   РЕЗУЛЬТАТ 1. КОЛИЧЕСТВО КЛИЕНТОВ И ДОГОВОРОВ
   ============================================================ */

SELECT
      (SELECT COUNT(*) FROM #new_clients) AS new_clients
    , COUNT(*) AS contracts
    , SUM(CASE WHEN product_type = N'Вклады'
               THEN 1 ELSE 0 END) AS deposit_contracts
    , SUM(CASE WHEN product_type = N'НС'
               THEN 1 ELSE 0 END) AS ns_contracts
FROM #contracts;

/* ============================================================
   РЕЗУЛЬТАТ 2. ПОДНЕВНАЯ ДИНАМИКА, МЛРД РУБ.
   ============================================================ */

SELECT
      dt_rep
    , CAST(balance_deposits / 1000000000.0
           AS decimal(18,6)) AS deposits_bn
    , CAST(balance_ns / 1000000000.0
           AS decimal(18,6)) AS ns_bn
    , CAST((balance_deposits + balance_ns) / 1000000000.0
           AS decimal(18,6)) AS total_bn
FROM #daily
ORDER BY dt_rep;

/* ============================================================
   РЕЗУЛЬТАТ 3. ОСТАТКИ НА 7 АПРЕЛЯ, МЛРД РУБ.
   ============================================================ */

SELECT
      dt_rep
    , CAST(balance_deposits / 1000000000.0
           AS decimal(18,6)) AS deposits_bn
    , CAST(balance_ns / 1000000000.0
           AS decimal(18,6)) AS ns_bn
    , CAST((balance_deposits + balance_ns) / 1000000000.0
           AS decimal(18,6)) AS total_bn
FROM #daily
WHERE dt_rep = @dt_to;
