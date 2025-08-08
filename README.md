USE ALM_TEST;
GO

/* ═════ ПАРАМЕТРЫ ═════ */
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.065;

/* ═════ 1. КАЛЕНДАРЬ И КЛЮЧЕВАЯ СТАВКА ═════ */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal)<@HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

DROP TABLE IF EXISTS #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* ═════ 2. ПРОМО-ПОРТФЕЛЬ (ИЮЛЬ–АВГУСТ) ═════ */
DROP TABLE IF EXISTS #bal_prom;
SELECT t.con_id, t.cli_id, t.out_rub, t.rate_con,
       CAST(t.dt_open AS date) dt_open,
       t.TSEGMENTNAME
INTO   #bal_prom
FROM   ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE  t.dt_rep = @Anchor
  AND  t.prod_id = 654
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name = N'Привлечение ФЛ'
  AND  t.cur = '810' AND t.od_flag = 1
  AND  t.out_rub IS NOT NULL
  AND  t.dt_open BETWEEN '2025-07-01' AND '2025-08-31';

/* ═════ 3. СПРЕДЫ ПО АВГУСТУ ═════ */
DROP TABLE IF EXISTS WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads (
    TSEGMENTNAME nvarchar(40) PRIMARY KEY,
    spread       decimal(9,4)
);

DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

;WITH aug AS (
    SELECT TSEGMENTNAME,
           w_rate = SUM(out_rub*rate_con)/SUM(out_rub)
    FROM   #bal_prom
    WHERE  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
    GROUP  BY TSEGMENTNAME)
SELECT @Spread_DChbo  = MAX(CASE WHEN TSEGMENTNAME = N'ДЧБО'            THEN w_rate END)
                           - (SELECT KEY_RATE FROM #key WHERE DT_REP = @Anchor),
       @Spread_Retail = MAX(CASE WHEN TSEGMENTNAME = N'Розничный бизнес' THEN w_rate END)
                           - (SELECT KEY_RATE FROM #key WHERE DT_REP = @Anchor)
FROM   aug;

INSERT WORK.NS_Spreads VALUES
    (N'ДЧБО',            @Spread_DChbo),
    (N'Розничный бизнес',@Spread_Retail);

/* ═════ 4. ОКНА С ПЕРЕХОДОМ И cycle_id ═════ */
DROP TABLE IF EXISTS #promo_win;
;WITH seed AS (
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub,
           win_start = dt_open,
           win_end   = DATEADD(day, -1, EOMONTH(DATEADD(month, 1, dt_open))),
           cycle_id = CAST(con_id AS varchar) + '_0'
    FROM   #bal_prom
),
seq AS (
    SELECT * FROM seed
    UNION ALL
    SELECT s.con_id, s.cli_id, s.TSEGMENTNAME, s.out_rub,
           win_start = DATEADD(day, 1, s.win_end),
           win_end   = DATEADD(day, -1, EOMONTH(DATEADD(month, 1, DATEADD(day,1,s.win_end)))),
           cycle_id  = CAST(s.con_id AS varchar) + '_' + CAST(CAST(RIGHT(s.cycle_id, LEN(s.cycle_id) - CHARINDEX('_', s.cycle_id)) AS int) + 1 AS varchar)
    FROM   seq s
    WHERE  DATEADD(day, 1, s.win_end) <= @HorizonTo
)
SELECT * INTO #promo_win FROM seq OPTION (MAXRECURSION 0);

/* ═════ 5. RAW-ПРОМО СТАВКИ ═════ */
DROP TABLE IF EXISTS #FIX_promo_raw;
;WITH core AS (
    SELECT p.con_id, p.cli_id, p.TSEGMENTNAME, p.out_rub,
           c.d AS dt_rep,
           p.win_start, p.win_end,
           p.cycle_id,
           k1.KEY_RATE AS key_open,
           k2.KEY_RATE AS key_day
    FROM   #promo_win p
    JOIN   #cal c ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
    JOIN   #key k1 ON k1.DT_REP = p.win_start
    LEFT   JOIN #key k2 ON k2.DT_REP = c.d
)
SELECT con_id, cli_id, TSEGMENTNAME, out_rub,
       dt_rep, cycle_id,
       rate_con = CASE
         WHEN dt_rep < win_end THEN
            CASE TSEGMENTNAME
              WHEN N'ДЧБО' THEN @Spread_DChbo + key_open
              ELSE            @Spread_Retail + key_open END
         WHEN dt_rep = win_end THEN @BaseRate
         ELSE
            CASE TSEGMENTNAME
              WHEN N'ДЧБО' THEN @Spread_DChbo + key_day
              ELSE            @Spread_Retail + key_day END
       END
INTO #FIX_promo_raw
FROM core
WHERE NOT (DAY(dt_rep) = 1);  -- исключаем 1-е число, оно в glue

/* ═════ 6. ПЕРЕЛИВ НА 1-Е ЧИСЛО ═════ */
DROP TABLE IF EXISTS #FIX_promo_glue;
;WITH p1 AS (
    SELECT DISTINCT cli_id,
           m1 = DATEFROMPARTS(YEAR(win_end), MONTH(win_end), 1)
    FROM   #promo_win
)
SELECT
    CAST(NULL AS bigint)       AS con_id,
    d.cli_id,
    CAST(NULL AS nvarchar(40)) AS TSEGMENTNAME,
    SUM(out_rub)               AS out_rub,
    p.m1                       AS dt_rep,
    MAX(rate_con)              AS rate_con,
    MAX(cycle_id)              AS cycle_id
INTO #FIX_promo_glue
FROM   #FIX_promo_raw d
JOIN   p1 p ON p.cli_id = d.cli_id AND p.m1 = d.dt_rep
GROUP  BY d.cli_id, p.m1;

/* ═════ 7. ФИНАЛЬНАЯ ПРОМО-ЛЕНТА ═════ */
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;
SELECT dt_rep,
       SUM(out_rub) out_rub_total,
       SUM(out_rub*rate_con)/SUM(out_rub) rate_avg
INTO   WORK.Forecast_NS_Promo
FROM (
    SELECT * FROM #FIX_promo_raw
    UNION ALL
    SELECT * FROM #FIX_promo_glue
) x
GROUP BY dt_rep;

/* ═════ 8. ПРОМО СТАВКА 1-ГО ЧИСЛА ═════ */
DROP TABLE IF EXISTS WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates (
  month_first   date PRIMARY KEY,
  TSEGMENTNAME  nvarchar(40),
  promo_rate    decimal(9,4)
);
;WITH mkey AS (
    SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) m1,
           MIN(KEY_RATE) key_min
    FROM   #key
    GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1)
)
INSERT WORK.NS_PromoRates
SELECT m1, s.TSEGMENTNAME, m.key_min + s.spread
FROM   mkey m
JOIN   WORK.NS_Spreads s ON 1=1;

/* ═════ 9. КОНТРОЛЬНЫЕ ВЫВОДЫ ═════ */
PRINT N'=== spread (Aug-25) ===';         SELECT * FROM WORK.NS_Spreads;
PRINT N'=== promo-rate (1-е) ===';        SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,TSEGMENTNAME;
PRINT N'=== promo-лента (TOP-40) ===';    SELECT TOP (40) * FROM WORK.Forecast_NS_Promo ORDER BY dt_rep;

GO
