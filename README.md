USE [ALM];
SET NOCOUNT ON;

-------------------------------------------------------
-- Параметры периода
-------------------------------------------------------
DECLARE @StartDate date = '2023-08-01';  -- начиная с августа 2023
DECLARE @EndDate   date = '2025-11-30';  -- до конца ноября 2025

-------------------------------------------------------
-- 1. Календарь "концов месяца" по фактическим снимкам
-------------------------------------------------------
IF OBJECT_ID('tempdb..#Calendar') IS NOT NULL
    DROP TABLE #Calendar;

;WITH MonthMax AS (
    SELECT
          YEAR(dt_rep)  AS y
        , MONTH(dt_rep) AS m
        , MAX(dt_rep)   AS dt_rep
    FROM [ALM].[ALM].[VW_balance_rest_all] v WITH (NOLOCK)
    WHERE v.dt_rep BETWEEN @StartDate AND @EndDate
    GROUP BY YEAR(dt_rep), MONTH(dt_rep)
)
SELECT
      mm.dt_rep                                            AS dt_rep              -- фактический снимок
    , DATEFROMPARTS(mm.y, mm.m, 1)                         AS month_start         -- первое число месяца
    , EOMONTH(DATEFROMPARTS(mm.y, mm.m, 1))                AS month_end           -- календарный конец месяца
INTO #Calendar
FROM MonthMax mm;

-------------------------------------------------------
-- 2. Агрегация по портфелю
-------------------------------------------------------
SELECT
      c.dt_rep

    -- б) весь рублёвый портфель ФЛ на эту дату
    , portfolio_volume =
        SUM(t.out_rub)

    -- в) объём портфеля, открытого в этом месяце (по dt_open)
    , opened_volume =
        SUM(
            CASE
                WHEN t.dt_open >= c.month_start
                 AND t.dt_open <= c.month_end
                THEN t.out_rub
                ELSE 0
            END
        )

    -- г) средневзвешенная ставка по открытым в этом месяце
    --    WAVG только по строкам с нормальной ставкой и объёмом
    , client_rate_of_opened =
        SUM(
            CASE
                WHEN t.dt_open >= c.month_start
                 AND t.dt_open <= c.month_end
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
                     AND t.dt_open <= c.month_end
                     AND t.rate_con IS NOT NULL
                     AND t.rate_con > 0
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
    AND ISNULL(t.prod_name_res, N'') NOT IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2',
           N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный',
           N'ДОМа надёжно', N'Всё в ДОМ', N'Могучий'
       )
GROUP BY
    c.dt_rep
ORDER BY
    c.dt_rep;
