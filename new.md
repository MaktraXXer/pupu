USE [ALM];
SET NOCOUNT ON;

/* =========================
   ПАРАМЕТРЫ
   ========================= */
DECLARE @BaseDate date = '2026-03-31';
DECLARE @EndDate  date = '2026-04-13';

DECLARE @ExitFrom date = '2026-04-01';
DECLARE @ExitTo   date = '2026-04-13';

DECLARE @OpenFrom date = '2026-04-01';
DECLARE @OpenTo   date = '2026-04-13';

DECLARE @Promo1From date = '2026-04-01';
DECLARE @Promo1To   date = '2026-04-07';

DECLARE @Promo2From date = '2026-04-08';
DECLARE @Promo2To   date = '2026-04-13';

DECLARE @eps decimal(9,6) = 0.0005;

IF OBJECT_ID('tempdb..#rk_rates') IS NOT NULL DROP TABLE #rk_rates;
IF OBJECT_ID('tempdb..#bal_base') IS NOT NULL DROP TABLE #bal_base;
IF OBJECT_ID('tempdb..#bal_end') IS NOT NULL DROP TABLE #bal_end;
IF OBJECT_ID('tempdb..#client_scope') IS NOT NULL DROP TABLE #client_scope;
IF OBJECT_ID('tempdb..#client_mart') IS NOT NULL DROP TABLE #client_mart;

/* =========================
   ПРОМО-СТАВКИ
   ========================= */
CREATE TABLE #rk_rates
(
      promo_group nvarchar(20)
    , d_from date
    , d_to date
    , conv_type varchar(20)
    , r decimal(9,6)
);

INSERT INTO #rk_rates(promo_group, d_from, d_to, conv_type, r)
VALUES
(N'promo_1', @Promo1From, @Promo1To, 'AT_THE_END',     0.160),
(N'promo_1', @Promo1From, @Promo1To, 'AT_THE_END',     0.158),
(N'promo_1', @Promo1From, @Promo1To, 'NOT_AT_THE_END', 0.158),
(N'promo_1', @Promo1From, @Promo1To, 'NOT_AT_THE_END', 0.156),

(N'promo_2', @Promo2From, @Promo2To, 'AT_THE_END',     0.150),
(N'promo_2', @Promo2From, @Promo2To, 'AT_THE_END',     0.148),
(N'promo_2', @Promo2From, @Promo2To, 'NOT_AT_THE_END', 0.148),
(N'promo_2', @Promo2From, @Promo2To, 'NOT_AT_THE_END', 0.146);

/* =========================
   БАЛАНС НА СТАРТЕ
   ========================= */
SELECT
      CAST(t.dt_rep AS date) AS dt_rep
    , CAST(t.cli_id AS bigint) AS cli_id
    , CAST(t.con_id AS bigint) AS con_id
    , CAST(t.dt_open AS date) AS dt_open
    , CAST(t.dt_close_plan AS date) AS dt_close_plan
    , t.section_name
    , CAST(t.out_rub AS decimal(38,6)) AS out_rub
    , CAST(t.rate_con AS decimal(18,6)) AS rate_con
    , t.conv
    , t.termdays
    , t.TSEGMENTNAME
INTO #bal_base
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep = @BaseDate
  AND t.section_name IN (N'Срочные', N'Накопительный счёт')
  AND t.block_name = N'Привлечение ФЛ'
  AND t.acc_role   = N'LIAB'
  AND t.od_flag    = 1
  AND t.cur        = '810'
  AND t.out_rub IS NOT NULL
  AND t.out_rub >= 0;

/* =========================
   БАЛАНС НА КОНЕЦ
   ========================= */
SELECT
      CAST(t.dt_rep AS date) AS dt_rep
    , CAST(t.cli_id AS bigint) AS cli_id
    , CAST(t.con_id AS bigint) AS con_id
    , CAST(t.dt_open AS date) AS dt_open
    , CAST(t.dt_close_plan AS date) AS dt_close_plan
    , t.section_name
    , CAST(t.out_rub AS decimal(38,6)) AS out_rub
    , CAST(t.rate_con AS decimal(18,6)) AS rate_con
    , t.conv
    , t.termdays
    , t.TSEGMENTNAME
INTO #bal_end
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep = @EndDate
  AND t.section_name IN (N'Срочные', N'Накопительный счёт')
  AND t.block_name = N'Привлечение ФЛ'
  AND t.acc_role   = N'LIAB'
  AND t.od_flag    = 1
  AND t.cur        = '810'
  AND t.out_rub IS NOT NULL
  AND t.out_rub >= 0;

/* =========================
   КЛИЕНТЫ С ВКЛАДАМИ К ВЫХОДУ
   ========================= */
SELECT DISTINCT cli_id
INTO #client_scope
FROM #bal_base
WHERE section_name = N'Срочные'
  AND dt_close_plan >= @ExitFrom
  AND dt_close_plan <= @ExitTo;

/* =========================
   ПОКЛИЕНТНАЯ ВИТРИНА
   ========================= */
WITH client_flags AS
(
    SELECT
          c.cli_id
        , CASE
              WHEN EXISTS (
                  SELECT 1
                  FROM #bal_base b
                  WHERE b.cli_id = c.cli_id
                    AND b.section_name IN (N'Срочные', N'Накопительный счёт')
                    AND b.TSEGMENTNAME = N'ДЧБО'
              )
              THEN N'ДЧБО'
              ELSE N'Розница'
          END AS client_segment
    FROM #client_scope c
),
exit_sum AS
(
    SELECT
          cli_id
        , SUM(out_rub) AS exit_td_sum
    FROM #bal_base
    WHERE section_name = N'Срочные'
      AND dt_close_plan >= @ExitFrom
      AND dt_close_plan <= @ExitTo
    GROUP BY cli_id
),
ns_start AS
(
    SELECT
          cli_id
        , SUM(out_rub) AS ns_start_sum
    FROM #bal_base
    WHERE section_name = N'Накопительный счёт'
    GROUP BY cli_id
),
ns_end AS
(
    SELECT
          cli_id
        , SUM(out_rub) AS ns_end_sum
    FROM #bal_end
    WHERE section_name = N'Накопительный счёт'
    GROUP BY cli_id
),
opened_by_con AS
(
    SELECT
          b.cli_id
        , b.con_id
        , SUM(b.out_rub) AS out_rub
        , MIN(b.rate_con) AS rate_con_class
        , MIN(b.termdays) AS termdays
        , MIN(b.dt_open) AS dt_open
        , CASE
              WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(b.conv,''))), '')) IS NULL
                  THEN 'AT_THE_END'
              ELSE UPPER(LTRIM(RTRIM(MIN(b.conv))))
          END AS conv_norm
    FROM #bal_end b
    INNER JOIN #client_scope c
        ON b.cli_id = c.cli_id
    WHERE b.section_name = N'Срочные'
      AND b.dt_open >= @OpenFrom
      AND b.dt_open <= @OpenTo
    GROUP BY b.cli_id, b.con_id
),
opened_classified AS
(
    SELECT
          o.cli_id
        , o.con_id
        , o.out_rub
        , CASE
              WHEN o.termdays >= 45 AND o.termdays <= 79
               AND EXISTS (
                    SELECT 1
                    FROM #rk_rates r
                    WHERE r.promo_group = N'promo_1'
                      AND o.dt_open BETWEEN r.d_from AND r.d_to
                      AND (
                            (o.conv_norm = 'AT_THE_END' AND r.conv_type = 'AT_THE_END')
                         OR (o.conv_norm <> 'AT_THE_END' AND r.conv_type = 'NOT_AT_THE_END')
                          )
                      AND ABS(o.rate_con_class - r.r) <= @eps
               )
              THEN N'promo_1'

              WHEN o.termdays >= 45 AND o.termdays <= 79
               AND EXISTS (
                    SELECT 1
                    FROM #rk_rates r
                    WHERE r.promo_group = N'promo_2'
                      AND o.dt_open BETWEEN r.d_from AND r.d_to
                      AND (
                            (o.conv_norm = 'AT_THE_END' AND r.conv_type = 'AT_THE_END')
                         OR (o.conv_norm <> 'AT_THE_END' AND r.conv_type = 'NOT_AT_THE_END')
                          )
                      AND ABS(o.rate_con_class - r.r) <= @eps
               )
              THEN N'promo_2'

              ELSE N'base'
          END AS open_category
    FROM opened_by_con o
),
opened_agg AS
(
    SELECT
          cli_id
        , SUM(CASE WHEN open_category = N'base'    THEN out_rub ELSE 0 END) AS opened_base
        , SUM(CASE WHEN open_category = N'promo_2' THEN out_rub ELSE 0 END) AS opened_promo_2
        , SUM(CASE WHEN open_category = N'promo_1' THEN out_rub ELSE 0 END) AS opened_promo_1
        , SUM(out_rub) AS opened_total
    FROM opened_classified
    GROUP BY cli_id
)
SELECT
      c.cli_id
    , f.client_segment AS segment_flag

    , CASE
          WHEN ISNULL(e.exit_td_sum,0) < 1500000 THEN N'01. Выход < 1.5 млн'
          WHEN ISNULL(e.exit_td_sum,0) < 5000000 THEN N'02. Выход 1.5-5 млн'
          ELSE N'03. Выход >= 5 млн'
      END AS exit_amount_flag

    , ISNULL(e.exit_td_sum,0) AS exit_td_sum

    , ISNULL(ns1.ns_start_sum,0) AS ns_start_sum

    , ISNULL(o.opened_base,0) AS opened_base
    , ISNULL(o.opened_promo_2,0) AS opened_promo_2_from_8_apr
    , ISNULL(o.opened_promo_1,0) AS opened_promo_1_from_1_7_apr

    , ISNULL(ns2.ns_end_sum,0) AS ns_end_sum
    , ISNULL(ns2.ns_end_sum,0) - ISNULL(ns1.ns_start_sum,0) AS ns_delta

    /* 1. Все открытые вклады / выходящие вклады */
    , CAST(
        ISNULL(o.opened_total,0)
        / NULLIF(ISNULL(e.exit_td_sum,0),0)
      AS decimal(18,6)) AS retention_1_td_all

    /* 2. Дельта НС + все открытые вклады / выходящие вклады */
    , CAST(
        (
            ISNULL(ns2.ns_end_sum,0) - ISNULL(ns1.ns_start_sum,0)
            + ISNULL(o.opened_total,0)
        )
        / NULLIF(ISNULL(e.exit_td_sum,0),0)
      AS decimal(18,6)) AS retention_2_td_all_plus_ns

    /* 3. База + промо с 8 апреля / выходящие вклады */
    , CAST(
        (
            ISNULL(o.opened_base,0)
            + ISNULL(o.opened_promo_2,0)
        )
        / NULLIF(ISNULL(e.exit_td_sum,0),0)
      AS decimal(18,6)) AS retention_3_td_base_plus_promo2

    /* 4. Дельта НС + база + промо с 8 апреля / выходящие вклады */
    , CAST(
        (
            ISNULL(ns2.ns_end_sum,0) - ISNULL(ns1.ns_start_sum,0)
            + ISNULL(o.opened_base,0)
            + ISNULL(o.opened_promo_2,0)
        )
        / NULLIF(ISNULL(e.exit_td_sum,0),0)
      AS decimal(18,6)) AS retention_4_td_base_plus_promo2_plus_ns

    /* 5. Только база / выходящие вклады */
    , CAST(
        ISNULL(o.opened_base,0)
        / NULLIF(ISNULL(e.exit_td_sum,0),0)
      AS decimal(18,6)) AS retention_5_td_base

    /* 6. Дельта НС + только база / выходящие вклады */
    , CAST(
        (
            ISNULL(ns2.ns_end_sum,0) - ISNULL(ns1.ns_start_sum,0)
            + ISNULL(o.opened_base,0)
        )
        / NULLIF(ISNULL(e.exit_td_sum,0),0)
      AS decimal(18,6)) AS retention_6_td_base_plus_ns

INTO #client_mart
FROM #client_scope c
LEFT JOIN client_flags f
    ON c.cli_id = f.cli_id
LEFT JOIN exit_sum e
    ON c.cli_id = e.cli_id
LEFT JOIN ns_start ns1
    ON c.cli_id = ns1.cli_id
LEFT JOIN ns_end ns2
    ON c.cli_id = ns2.cli_id
LEFT JOIN opened_agg o
    ON c.cli_id = o.cli_id;

/* =========================
   ПОКЛИЕНТНАЯ ВИТРИНА
   ========================= */
SELECT
      cli_id
    , segment_flag
    , exit_amount_flag
    , exit_td_sum
    , ns_start_sum
    , opened_base
    , opened_promo_2_from_8_apr
    , opened_promo_1_from_1_7_apr
    , ns_end_sum
    , ns_delta
    , retention_1_td_all
    , retention_2_td_all_plus_ns
    , retention_3_td_base_plus_promo2
    , retention_4_td_base_plus_promo2_plus_ns
    , retention_5_td_base
    , retention_6_td_base_plus_ns
FROM #client_mart
ORDER BY
      segment_flag
    , exit_amount_flag
    , cli_id;
