/* ▀▀  ПАРАМЕТРЫ  ▀▀ */
USE ALM_TEST;
GO
DECLARE @Anchor    date = '2025-07-15',      -- последний факт
        @HorizonTo date = '2025-09-30';      -- конец прогноза

/* ▀▀  СПРАВОЧНИК СПРЕДОВ  ▀▀ */
IF OBJECT_ID('tempdb..#spread_ref') IS NOT NULL DROP TABLE #spread_ref;
CREATE TABLE #spread_ref (
    segment_name NVARCHAR(50),
    termdays_low INT,
    termdays_high INT,
    to_be_rate DECIMAL(9,4),
    avg_forecast_key DECIMAL(9,6)
);

-- Розничный Бизнес
INSERT INTO #spread_ref VALUES 
    (N'Розничный Бизнес', 51, 71, 18.20, 0.174918),
    (N'Розничный Бизнес', 80, 100, 17.90, 0.173098),
    (N'Розничный Бизнес', 110, 130, 18.10, 0.169306),
    (N'Розничный Бизнес', 180, 200, 17.50, 0.1637),
    (N'Розничный Бизнес', 255, 290, 16.10, 0.1563),
    (N'Розничный Бизнес', 355, 375, 16.10, 0.1502),
    (N'Розничный Бизнес', 540, 560, 14.30, 0.1421),
    (N'Розничный Бизнес', 715, 745, 14.10, 0.1378),
    (N'Розничный Бизнес', 1085, 1115, 13.60, 0.1332);

-- Другие сегменты
INSERT INTO #spread_ref VALUES 
    (N'Другие', 51, 71, 18.60, 0.174918),
    (N'Другие', 80, 100, 18.30, 0.173098),
    (N'Другие', 110, 130, 18.50, 0.169306),
    (N'Другие', 180, 200, 17.90, 0.1637),
    (N'Другие', 255, 290, 16.60, 0.1563),
    (N'Другие', 355, 375, 16.60, 0.1502),
    (N'Другие', 540, 560, 15.00, 0.1421),
    (N'Другие', 715, 745, 14.80, 0.1378),
    (N'Другие', 1085, 1115, 14.30, 0.1332);

/* ▀▀  КАЛЕНДАРЬ  ▀▀ */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal)<@HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* ▀▀  SPOT-KEY  ▀▀ */
IF OBJECT_ID('tempdb..#key_spot') IS NOT NULL DROP TABLE #key_spot;
SELECT fc.DT_REP, fc.KEY_RATE
INTO #key_spot
FROM WORK.ForecastKey_Cache fc
JOIN #cal c ON c.d = fc.DT_REP
WHERE fc.TERM = 1;

/* ▀▀  БАЗА + СПРЕДЫ  ▀▀ */
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
SELECT 
    t.con_id, t.out_rub, t.rate_con, t.is_floatrate,
    t.termdays, t.dt_open, t.dt_close,
    t.segment_name, t.convention,
    spread_float = CASE WHEN t.is_floatrate = 1 
                        THEN t.rate_con - ks.KEY_RATE END,
    spread_fix = CASE 
        WHEN t.is_floatrate = 0 AND ref.segment_name IS NOT NULL
        THEN 
            CASE 
                WHEN t.convention = 'AT_THE_END' 
                THEN ref.to_be_rate - (ref.avg_forecast_key * 100)
                ELSE 
                    /* Пересчет ставки в ежемесячную конвенцию */
                    100 * (12.0 * (POWER(
                        1.0 + (ref.to_be_rate/100.0) * (CASE WHEN t.termdays BETWEEN 21 AND 41 THEN 91 ELSE t.termdays END)/365.0, 
                        365.0/(CASE WHEN t.termdays BETWEEN 21 AND 41 THEN 91 ELSE t.termdays END * 12.0)
                    ) - 1.0) - ref.avg_forecast_key)
            END
        ELSE t.rate_con - fk_open.AVG_KEY_RATE 
    END
INTO #base
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
JOIN #key_spot ks ON ks.DT_REP = @Anchor
LEFT JOIN WORK.ForecastKey_Cache fk_open
    ON fk_open.DT_REP = t.dt_open 
    AND fk_open.TERM = t.termdays
LEFT JOIN #spread_ref ref 
    ON ref.segment_name = CASE 
        WHEN t.segment_name = N'Розничный Бизнес' THEN N'Розничный Бизнес' 
        ELSE N'Другие' 
    END
    AND t.termdays BETWEEN ref.termdays_low AND ref.termdays_high
WHERE t.dt_rep = @Anchor
    AND t.section_name = N'Срочные'
    AND t.block_name = N'Привлечение ФЛ'
    AND t.od_flag = 1  
    AND t.cur = '810' 
    AND t.out_rub IS NOT NULL;

/* ▀▀  ROLL-OVER (с переворотом 31±10 в 91 день) ▀▀ */
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;
;WITH seq AS(
    SELECT 
        con_id, out_rub, is_floatrate, termdays,
        spread_float, spread_fix,
        dt_open, n = 0,
        orig_term = termdays  -- сохраняем исходный срок
    FROM #base
    
    UNION ALL
    
    SELECT 
        s.con_id, s.out_rub, s.is_floatrate,
        termdays = CASE 
            WHEN s.orig_term BETWEEN 21 AND 41 THEN 91  -- переворот в 91 день
            ELSE s.termdays 
        END,
        s.spread_float, s.spread_fix,
        DATEADD(day, s.termdays, s.dt_open),
        n = s.n + 1,
        s.orig_term
    FROM seq s
    WHERE DATEADD(day, s.termdays, s.dt_open) <= @HorizonTo
)
SELECT 
    con_id, out_rub, is_floatrate, termdays,
    dt_open,
    dt_close = DATEADD(day, termdays, dt_open),
    spread_float, spread_fix
INTO #rolls
FROM seq 
OPTION (MAXRECURSION 0);

/* ▀▀  КОНТРАКТЫ ▀▀ */
IF OBJECT_ID('tempdb..#work') IS NOT NULL DROP TABLE #work;
SELECT * INTO #work FROM #rolls;

/* ▀▀  ПОСУТОЧНО  ▀▀ */
IF OBJECT_ID('tempdb..#daily') IS NOT NULL DROP TABLE #daily;
SELECT 
    c.d AS dt_rep,
    w.con_id, w.out_rub,
    rate_con = CASE
        WHEN w.is_floatrate = 1
            THEN ks.KEY_RATE + w.spread_float
        ELSE ISNULL(fko.AVG_KEY_RATE + w.spread_fix, w.spread_fix)
    END
INTO #daily
FROM #cal c
JOIN #work w
    ON c.d BETWEEN w.dt_open AND DATEADD(day, -1, w.dt_close)
LEFT JOIN #key_spot ks
    ON ks.DT_REP = c.d
LEFT JOIN WORK.ForecastKey_Cache fko
    ON fko.DT_REP = w.dt_open 
    AND fko.TERM = w.termdays;

/* ▀▀  ЦЕЛЕВЫЕ ТАБЛИЦЫ  ▀▀ */
IF OBJECT_ID('WORK.Forecast_BalanceDaily','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDaily (
        dt_rep DATE PRIMARY KEY,
        out_rub_total DECIMAL(20,2),
        rate_con DECIMAL(9,4)
    );
ELSE 
    TRUNCATE TABLE WORK.Forecast_BalanceDaily;

IF OBJECT_ID('WORK.Forecast_BalanceDeals','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDeals (
        dt_rep DATE,
        con_id BIGINT,
        out_rub DECIMAL(20,2),
        rate_con DECIMAL(9,4),
        CONSTRAINT PK_FBD PRIMARY KEY(dt_rep, con_id)
    );
ELSE 
    TRUNCATE TABLE WORK.Forecast_BalanceDeals;

/* ▀▀  АГРЕГАТ  ▀▀ */
INSERT INTO WORK.Forecast_BalanceDaily(dt_rep, out_rub_total, rate_con)
SELECT 
    dt_rep,
    SUM(out_rub),
    SUM(out_rub * rate_con) / NULLIF(SUM(out_rub), 0)
FROM #daily
GROUP BY dt_rep;

/* ▀▀  ДЕТАЛКА  ▀▀ */
INSERT INTO WORK.Forecast_BalanceDeals(dt_rep, con_id, out_rub, rate_con)
SELECT dt_rep, con_id, out_rub, rate_con
FROM #daily;
