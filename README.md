/* ============================================================
   СПИСОК ПРОДУКТОВ МАРКЕТПЛЕЙСОВ
   ============================================================ */
DECLARE @mp TABLE (prod_name nvarchar(100) PRIMARY KEY);
INSERT INTO @mp(prod_name)
VALUES
 (N'Надёжный прайм'), (N'Надёжный VIP'), (N'Надёжный премиум'),
 (N'Надёжный промо'), (N'Надёжный старт'),
 (N'Надёжный Т2'), (N'Надёжный T2'),
 (N'Надёжный Мегафон'), (N'Надёжный процент'),
 (N'Надёжный'), (N'ДОМа надёжно'), (N'Всё в ДОМ');

/* ============================================================
   ПАРАМЕТРЫ
   ============================================================ */
DECLARE @dt_rep1 date = '2025-11-23';  -- снимок для выходов
DECLARE @dt_rep2 date = '2025-12-06';  -- снимок для открытий
DECLARE @d_from  date = '2025-11-24';
DECLARE @d_to    date = '2025-12-06';

/* ============================================================
   ШАГ 1.
   МАРКЕТ-ВКЛАДЫ, КОТОРЫЕ ДОЛЖНЫ ВЫЙТИ 24–06 ДЕКАБРЯ
   ============================================================ */

IF OBJECT_ID('tempdb..#mp_exits') IS NOT NULL DROP TABLE #mp_exits;

SELECT
      t.cli_id,
      t.con_id,
      CAST(t.dt_close AS date) AS dt_close_d,
      t.out_rub
INTO #mp_exits
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep1
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.acc_role     = N'LIAB'
  AND t.cur          = '810'
  AND t.out_rub > 0
  AND t.dt_close > t.dt_rep
  AND CAST(t.dt_close AS date) BETWEEN @d_from AND @d_to
  AND EXISTS (SELECT 1 FROM @mp m WHERE m.prod_name = t.prod_name);

/* ============================================================
   ШАГ 1A — СПИСОК КЛИЕНТОВ
   ============================================================ */
IF OBJECT_ID('tempdb..#cli_mp_exit') IS NOT NULL DROP TABLE #cli_mp_exit;
SELECT DISTINCT cli_id INTO #cli_mp_exit FROM #mp_exits;

/* ============================================================
   === SELECT 1 ===
   ВЫХОДЫ МАРКЕТ-ВКЛАДОВ ПО ДАТАМ
   ============================================================ */
SELECT
      dt_close_d AS [date],
      COUNT(*)   AS cnt_market_exits,
      SUM(out_rub) AS sum_market_exits
FROM #mp_exits
GROUP BY dt_close_d
ORDER BY dt_close_d;

/* ============================================================
   ШАГ 2.
   СНИМОК НА 06.12: ОТКРЫТИЯ НЕ-МАРКЕТ ВКЛАДОВ ЭТИМИ КЛИЕНТАМИ
   ============================================================ */

IF OBJECT_ID('tempdb..#nonmp_open') IS NOT NULL DROP TABLE #nonmp_open;

SELECT
      t.cli_id,
      t.con_id,
      CAST(t.dt_open AS date) AS dt_open_d,
      t.out_rub
INTO #nonmp_open
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep   = @dt_rep2
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.acc_role     = N'LIAB'
  AND t.cur          = '810'
  AND t.cli_id IN (SELECT cli_id FROM #cli_mp_exit)
  AND t.out_rub > 0
  AND CAST(t.dt_open AS date) BETWEEN @d_from AND @d_to
  AND NOT EXISTS (SELECT 1 FROM @mp m WHERE m.prod_name = t.prod_name);

/* Убираем дубли вкладов */
;WITH agg_nonmp AS (
    SELECT
          dt_open_d,
          con_id,
          MIN(cli_id) AS cli_id,
          SUM(out_rub) AS out_rub
    FROM #nonmp_open
    GROUP BY dt_open_d, con_id
)
SELECT
      dt_open_d AS [date],
      COUNT(*)  AS cnt_nonmp_open,
      SUM(out_rub) AS sum_nonmp_open
FROM agg_nonmp
GROUP BY dt_open_d
ORDER BY dt_open_d;

/* ============================================================
   === SELECT 3 ===
   ОТКРЫТИЯ МАРКЕТ-ВКЛАДОВ ЭТИМИ КЛИЕНТАМИ
   ============================================================ */

IF OBJECT_ID('tempdb..#mp_open') IS NOT NULL DROP TABLE #mp_open;

SELECT
      t.cli_id,
      t.con_id,
      CAST(t.dt_open AS date) AS dt_open_d,
      t.out_rub
INTO #mp_open
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep   = @dt_rep2
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.acc_role     = N'LIAB'
  AND t.cur          = '810'
  AND t.cli_id IN (SELECT cli_id FROM #cli_mp_exit)
  AND t.out_rub > 0
  AND CAST(t.dt_open AS date) BETWEEN @d_from AND @d_to
  AND EXISTS (SELECT 1 FROM @mp m WHERE m.prod_name = t.prod_name);

/* Убираем дубли */
;WITH agg_mp AS (
    SELECT
          dt_open_d,
          con_id,
          MIN(cli_id) AS cli_id,
          SUM(out_rub) AS out_rub
    FROM #mp_open
    GROUP BY dt_open_d, con_id
)
SELECT
      dt_open_d AS [date],
      COUNT(*)  AS cnt_mp_open,
      SUM(out_rub) AS sum_mp_open
FROM agg_mp
GROUP BY dt_open_d
ORDER BY dt_open_d;

/* ============================================================
   === SELECT 4 ===
   ОСТАТКИ НА НАКОПИТЕЛЬНЫХ СЧЁТАХ ДЛЯ ЭТИХ ЖЕ КЛИЕНТОВ
   по снимкам 23.11 и 06.12
   ============================================================ */

SELECT
      '2025-11-23' AS dt_rep,
      SUM(t.out_rub) AS sum_ns_23
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep1
  AND t.section_name = N'Накопительный счёт'
  AND t.acc_role     = N'LIAB'
  AND t.cur          = '810'
  AND t.out_rub > 0
  AND t.cli_id IN (SELECT cli_id FROM #cli_mp_exit)

UNION ALL

SELECT
      '2025-12-06' AS dt_rep,
      SUM(t.out_rub) AS sum_ns_06
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep2
  AND t.section_name = N'Накопительный счёт'
  AND t.acc_role     = N'LIAB'
  AND t.cur          = '810'
  AND t.out_rub > 0
  AND t.cli_id IN (SELECT cli_id FROM #cli_mp_exit);
