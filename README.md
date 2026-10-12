USE ALM;
SET NOCOUNT ON;

DECLARE @DateOne DATE = '2026-09-30';
DECLARE @DateTwo DATE = '2026-10-03';
DECLARE @DateTri DATE = '2026-10-03';

;WITH Base AS (
    SELECT
        con_id,
        ISNULL(is_floatrate, 0) AS is_floatrate,
        SUM(out_rub) AS BalanceBefore
    FROM ALM.ALM.VW_balance_rest_all WITH (NOLOCK)
    WHERE dt_rep = @DateOne
      AND dt_close_plan > @DateTri
      AND section_name = N'Срочные'
      AND block_name = N'Привлечение ФЛ'
      AND od_flag = 1
      AND acc_role = 'LIAB'
      AND cur = '810'
      AND PROD_NAME_res NOT IN (
          N'Надёжный прайм', N'Надёжный VIP',
          N'Надёжный премиум', N'Надёжный промо',
          N'Надёжный старт', N'Надёжный Т2',
          N'Надёжный Мегафон', N'Надёжный процент',
          N'Могучий', N'Надёжный',
          N'ДОМа надёжно', N'Всё в ДОМ'
      )
    GROUP BY con_id, ISNULL(is_floatrate, 0)
),
BalanceAfter AS (
    SELECT
        t.con_id,
        SUM(t.out_rub) AS BalanceAfter
    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
    WHERE t.dt_rep = @DateTwo
      AND t.section_name = N'Срочные'
      AND t.block_name = N'Привлечение ФЛ'
      AND t.od_flag = 1
      AND t.acc_role = 'LIAB'
      AND t.cur = '810'
      AND EXISTS (
          SELECT 1
          FROM Base b
          WHERE b.con_id = t.con_id
      )
    GROUP BY t.con_id
)
SELECT
    CASE WHEN b.is_floatrate = 0
         THEN N'Фиксированные'
         ELSE N'Динамические'
    END AS RateType,

    SUM(b.BalanceBefore) AS BalanceBefore,

    SUM(ISNULL(a.BalanceAfter, 0)) AS BalanceAfter,

    SUM(ISNULL(a.BalanceAfter, 0) - b.BalanceBefore)
        AS BalanceChange,

    SUM(CASE WHEN a.con_id IS NULL
             THEN b.BalanceBefore ELSE 0 END)
        AS BalanceDisappeared,

    SUM(CASE WHEN a.con_id IS NULL
             THEN 1 ELSE 0 END)
        AS CountDisappeared

FROM Base b
LEFT JOIN BalanceAfter a
    ON b.con_id = a.con_id

GROUP BY b.is_floatrate
ORDER BY b.is_floatrate;
