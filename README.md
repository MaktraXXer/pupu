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
DECLARE @dt_rep1   date = '2025-11-23';  -- снимок 23 ноября
DECLARE @dt_rep2   date = '2025-12-06';  -- снимок 6 декабря
DECLARE @d_from    date = '2025-11-24';  -- окно 24.11–06.12
DECLARE @d_to      date = '2025-12-06';

/* ============================================================
   ШАГ 1.
   КЛИЕНТЫ, У КОТОРЫХ В СНИМКЕ 23.11.2025
   С 24.11 ПО 06.12 ПЛАНОВО ВЫХОДЯТ ВКЛАДЫ МАРКЕТПЛЕЙСОВ
   ============================================================ */
IF OBJECT_ID('tempdb..#cli_mp_exit') IS NOT NULL DROP TABLE #cli_mp_exit;

SELECT DISTINCT
      t.cli_id
INTO #cli_mp_exit
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep1
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.acc_role     = N'LIAB'
  AND t.cur          = '810'
  AND t.out_rub IS NOT NULL AND t.out_rub >= 0
  AND t.dt_close > t.dt_rep
  AND CAST(t.dt_close AS date) BETWEEN @d_from AND @d_to
  AND EXISTS (SELECT 1 FROM @mp m WHERE m.prod_name = t.prod_name);

/* ============================================================
   ШАГ 2.
   СНИМОК НА 06.12.2025:
   ВКЛАДЫ НЕ МАРКЕТПЛЕЙСОВ, ОТКРЫТЫЕ 24.11–06.12 ЭТИМИ КЛИЕНТАМИ
   ============================================================ */
IF OBJECT_ID('tempdb..#base_open') IS NOT NULL DROP TABLE #base_open;

SELECT
      t.cli_id,
      t.con_id,
      CAST(t.dt_open AS date) AS dt_open_d,
      t.out_rub
INTO #base_open
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep2
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.acc_role     = N'LIAB'
  AND t.cur          = '810'
  AND t.out_rub IS NOT NULL AND t.out_rub >= 0
  AND t.cli_id IN (SELECT cli_id FROM #cli_mp_exit)
  AND CAST(t.dt_open AS date) BETWEEN @d_from AND @d_to
  AND NOT EXISTS (SELECT 1 FROM @mp m WHERE m.prod_name = t.prod_name);

/* На всякий случай убираем дубли по одному договору в снимке */
;WITH by_con AS (
    SELECT
          dt_open_d,
          con_id,
          MIN(cli_id)  AS cli_id,
          SUM(out_rub) AS out_rub
    FROM #base_open
    GROUP BY dt_open_d, con_id
)
SELECT
      dt_open_d      AS [open_date],
      COUNT(*)       AS cnt_deposits_open,
      SUM(out_rub)   AS sum_out_rub_open
FROM by_con
GROUP BY dt_open_d
ORDER BY dt_open_d;
