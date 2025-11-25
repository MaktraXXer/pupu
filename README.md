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
   ПАРАМЕТР
   ============================================================ */
DECLARE @dt_rep date = '2025-11-23';

/* ============================================================
   ШАГ 1.
   КЛИЕНТЫ, У КОТОРЫХ В ЭТОМ СНИМКЕ ЕСТЬ ЖИВЫЕ МАРКЕТПЛЕЙС-ВКЛАДЫ
   ============================================================ */

IF OBJECT_ID('tempdb..#cli_mp_now') IS NOT NULL DROP TABLE #cli_mp_now;

SELECT DISTINCT
      t.cli_id
INTO #cli_mp_now
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.acc_role     = N'LIAB'
  AND t.cur          = '810'
  AND t.out_rub IS NOT NULL AND t.out_rub >= 0
  AND t.dt_close > t.dt_rep                -- живые
  AND EXISTS (SELECT 1 FROM @mp m WHERE m.prod_name = t.prod_name);

/* ============================================================
   ШАГ 2.
   СНИМОК ЭТИХ ЖЕ КЛИЕНТОВ, НО ВКЛАДЫ — НЕ МАРКЕТПЛЕЙСЫ
   ============================================================ */

IF OBJECT_ID('tempdb..#base2') IS NOT NULL DROP TABLE #base2;

SELECT
      t.cli_id,
      t.con_id,
      CAST(t.dt_close AS date) AS dt_close_d,
      t.out_rub
INTO #base2
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.acc_role     = N'LIAB'
  AND t.cur          = '810'
  AND t.out_rub IS NOT NULL AND t.out_rub >= 0
  AND t.dt_close > t.dt_rep                    -- живые
  AND t.cli_id IN (SELECT cli_id FROM #cli_mp_now)
  AND NOT EXISTS (SELECT 1 FROM @mp m WHERE m.prod_name = t.prod_name);

/* ============================================================
   ШАГ 3.
   КАЛЕНДАРЬ ДО МАКСИМАЛЬНОГО dt_close
   ============================================================ */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;

DECLARE @d_end date;
SELECT @d_end = ISNULL(MAX(dt_close_d), @dt_rep) FROM #base2;

;WITH cal AS (
    SELECT @dt_rep AS d
    UNION ALL
    SELECT DATEADD(DAY, 1, d)
    FROM cal
    WHERE d < @d_end
)
SELECT d INTO #cal FROM cal OPTION (MAXRECURSION 0);

/* ============================================================
   ШАГ 4.
   ИТОГОВЫЙ ВЫВОД:
   ТОЛЬКО КОЛ-ВО ВЫХОДОВ И СУММА ВЫХОДОВ
   ============================================================ */

SELECT
      c.d AS [date],
      COUNT(CASE WHEN b.dt_close_d = c.d THEN 1 END) AS cnt_deposits_exit,
      SUM(CASE WHEN b.dt_close_d = c.d THEN b.out_rub END) AS sum_out_rub_exit
FROM #cal c
LEFT JOIN #base2 b
       ON b.dt_close_d = c.d
GROUP BY c.d
ORDER BY c.d;
