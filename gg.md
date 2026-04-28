USE [ALM];
SET NOCOUNT ON;

DECLARE @StartDate date = '2026-04-01';
DECLARE @EndDate   date = '2026-04-23';


IF OBJECT_ID('tempdb..#Calendar') IS NOT NULL
    DROP TABLE #Calendar;


;WITH Months AS (
    SELECT DATEFROMPARTS(YEAR(@StartDate), MONTH(@StartDate), 1) AS month_start

    UNION ALL

    SELECT DATEADD(MONTH, 1, month_start)
    FROM Months
    WHERE DATEADD(MONTH, 1, month_start) <= DATEFROMPARTS(YEAR(@EndDate), MONTH(@EndDate), 1)
)
SELECT
      dt_rep =
          CASE
              WHEN EOMONTH(month_start) > @EndDate
              THEN @EndDate
              ELSE EOMONTH(month_start)
          END
    , month_start = month_start
    , month_end_for_opened =
          CASE
              WHEN EOMONTH(month_start) > @EndDate
              THEN @EndDate
              ELSE EOMONTH(month_start)
          END
INTO #Calendar
FROM Months
OPTION (MAXRECURSION 300);


SELECT
      c.dt_rep

    , portfolio_volume =
        SUM(t.out_rub)

    , opened_volume =
        SUM(
            CASE
                WHEN t.dt_open >= c.month_start
                 AND t.dt_open <= c.month_end_for_opened
                THEN t.out_rub
                ELSE 0
            END
        )

    , client_rate_of_opened =
        SUM(
            CASE
                WHEN t.dt_open >= c.month_start
                 AND t.dt_open <= c.month_end_for_opened
                 AND t.rate_con IS NOT NULL
                 AND t.rate_con > 0
                 AND t.out_rub  > 0
                THEN t.out_rub * t.rate_con
                ELSE 0
            END
        )
        /
        NULLIF(
            SUM(
                CASE
                    WHEN t.dt_open >= c.month_start
                     AND t.dt_open <= c.month_end_for_opened
                     AND t.rate_con IS NOT NULL
                     AND t.rate_con > 0
                     AND t.out_rub  > 0
                    THEN t.out_rub
                    ELSE 0
                END
            ),
            0
        )

    , rate_trf_of_opened =
        SUM(
            CASE
                WHEN t.dt_open >= c.month_start
                 AND t.dt_open <= c.month_end_for_opened
                 AND t.rate_trf IS NOT NULL
                 AND t.rate_trf > 0
                 AND t.out_rub  > 0
                THEN t.out_rub * t.rate_trf
                ELSE 0
            END
        )
        /
        NULLIF(
            SUM(
                CASE
                    WHEN t.dt_open >= c.month_start
                     AND t.dt_open <= c.month_end_for_opened
                     AND t.rate_trf IS NOT NULL
                     AND t.rate_trf > 0
                     AND t.out_rub  > 0
                    THEN t.out_rub
                    ELSE 0
                END
            ),
            0
        )

FROM #Calendar c
LEFT JOIN [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
    ON  t.dt_rep = c.dt_rep
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag      = 1
    AND t.cur          = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
    AND t.AP = N'Пассив'

/*
    AND ISNULL(t.prod_name_res, N'') NOT IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2',
           N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный',
           N'ДОМа надёжно', N'Всё в ДОМ', N'Могучий'
       )
*/

GROUP BY
    c.dt_rep

ORDER BY
    c.dt_rep;
