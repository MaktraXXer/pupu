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
   ШАГ 1.
   КЛИЕНТЫ, У КОТОРЫХ ПЕРВЫЙ ВКЛАД В ЖИЗНИ — МАРКЕТПЛЕЙС
   ============================================================ */

IF OBJECT_ID('tempdb..#hui') IS NOT NULL DROP TABLE #hui;

;WITH base AS (
    SELECT
          a.cli_id,
          a.con_id,
          a.prod_name,
          a.dt_open
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    WHERE a.dt_open IS NOT NULL
      AND a.cli_short_name = N'ФЛ'
      AND a.seg_name       = N'Розничный бизнес'
      AND a.cur            = 'RUR'
),
first_deal AS (
    SELECT
          b.*,
          ROW_NUMBER() OVER (PARTITION BY b.cli_id ORDER BY b.dt_open, b.con_id) AS rn
    FROM base b
),
mp_origin AS (
    SELECT cli_id
    FROM first_deal f
    WHERE f.rn = 1
      AND EXISTS (SELECT 1 FROM @mp m WHERE m.prod_name = f.prod_name)
)
SELECT DISTINCT cli_id INTO #hui FROM mp_origin;

/* ============================================================
   ШАГ 2.
   СНИМОК НА 23.11.2025 (ТОЛЬКО НЕ-МАРКЕТПЛЕЙСОВЫЕ ВКЛАДЫ)
   ============================================================ */

DECLARE @dt_rep date = '2025-11-23';

IF OBJECT_ID('tempdb..#base2') IS NOT NULL DROP TABLE #base2;

SELECT
      t.cli_id,
      t.con_id,
      t.cur,
      CAST(t.dt_close AS date) AS dt_close_d,
      t.out_rub,
      t.prod_name
INTO #base2
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.acc_role     = N'LIAB'
  AND t.cur          = '810'
  AND t.out_rub IS NOT NULL
  AND t.out_rub >= 0
  AND t.cli_id IN (SELECT cli_id FROM #hui)
  AND NOT EXISTS (SELECT 1 FROM @mp m WHERE m.prod_name = t.prod_name)
  AND t.dt_close > t.dt_rep;

/* ============================================================
   ШАГ 3.
   КАЛЕНДАРЬ ДО МАКСИМАЛЬНОЙ ДАТЫ ЗАКРЫТИЯ
   ============================================================ */

IF OBJECT_ID('tempdb..#cal2') IS NOT NULL DROP TABLE #cal2;

DECLARE @d_end date;
SELECT @d_end = ISNULL(MAX(dt_close_d), @dt_rep) FROM #base2;

;WITH cal AS (
    SELECT @dt_rep AS d
    UNION ALL
    SELECT DATEADD(DAY,1,d)
    FROM cal
    WHERE d < @d_end
)
SELECT d INTO #cal2 FROM cal OPTION (MAXRECURSION 0);

/* ============================================================
   ШАГ 4.
   ИТОГ — ТОЛЬКО КОЛ-ВО И СУММА ВЫХОДОВ ПО ДАТАМ
   ============================================================ */

SELECT
      c.d AS [date],
      COUNT(CASE WHEN b.dt_close_d = c.d THEN 1 END) AS cnt_deposits_exit,
      SUM(CASE WHEN b.dt_close_d = c.d THEN b.out_rub END) AS sum_out_rub_exit
FROM #cal2 c
LEFT JOIN #base2 b
       ON b.dt_close_d = c.d
GROUP BY c.d
ORDER BY c.d;
