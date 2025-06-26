/* ► ваш прежний «эталон» */
WITH old_tot AS (
    SELECT [Date],
           SUM(BALANCE_RUB) AS bal_old
    FROM   ALM_TEST.WORK.test_newUL_cube   -- ваша старая витрина
    GROUP  BY [Date]
),

/* ► новый CUBE, но только тотал-строка
   (оба флага агрегированы) */
     new_tot AS (
    SELECT [Date],
           SUM(BALANCE_RUB) AS bal_new
    FROM   ALM_TEST.WORK.test_UL_cube_full   -- новая витрина
    WHERE  IS_PDR        = 'all PDR'
      AND  IS_FINANCE_LCR= 'all FINANCE_LCR'
      AND  TERM_GROUP    = 'all termgroup'
      AND  IS_OPTION     = 'all option_type'
      AND  SEG_NAME      = 'all segment'
      AND  MARGIN_TYPE   = 'all margin'
      AND  CLI_SUBTYPE   = 'ЮЛ'              -- агрегир. уровень
      AND  [TYPE]        = 'Начало'
    GROUP BY [Date]
)
SELECT o.[Date], o.bal_old, n.bal_new,
       n.bal_new - o.bal_old AS diff
FROM   old_tot o
FULL   JOIN new_tot n ON n.[Date] = o.[Date]
ORDER  BY 1;
