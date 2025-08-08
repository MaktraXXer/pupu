/* ══════════════════════════════════════════════════════════════
   NS-forecast — Part 2 (FIX-promo-roll, prod_id = 654)
   Полностью переписанная логика: cycle_id, денормализация, без glue
   v.2025-08-09
═══════════════════════════════════════════════════════════════*/

USE ALM_TEST;
GO

/* ── параметры ─────────────────────────────────────────────── */
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/* ── 0. Календарь и ключевая ставка ────────────────────────── */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

DROP TABLE IF EXISTS #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* ── 1. Снимок promo (только июль–август-25) ───────────────── */
DROP TABLE IF EXISTS #bal_prom;
SELECT  t.con_id, t.cli_id, t.out_rub, t.rate_con,
        CAST(t.dt_open AS date) dt_open,
        t.TSEGMENTNAME
INTO    #bal_prom
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE   t.dt_rep=@Anchor
  AND   t.prod_id = 654
  AND   t.section_name = N'Накопительный счёт'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.cur = '810' AND t.od_flag=1
  AND   t.out_rub IS NOT NULL
  AND   t.dt_open BETWEEN '2025-07-01' AND '2025-08-31';

/* ── 2. Спреды по августовским открытиям ───────────────────── */
DROP TABLE IF EXISTS WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(
  TSEGMENTNAME nvarchar(40) PRIMARY KEY,
  spread       decimal(9,4));

DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

;WITH aug AS (
    SELECT TSEGMENTNAME,
           w_rate = SUM(out_rub * rate_con) / SUM(out_rub)
    FROM   #bal_prom
    WHERE  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
    GROUP BY TSEGMENTNAME)
SELECT @Spread_DChbo  = MAX(CASE WHEN TSEGMENTNAME = N'ДЧБО' THEN w_rate END)
                       - (SELECT KEY_RATE FROM #key WHERE DT_REP = @Anchor),
       @Spread_Retail = MAX(CASE WHEN TSEGMENTNAME = N'Розничный бизнес' THEN w_rate END)
                       - (SELECT KEY_RATE FROM #key WHERE DT_REP = @Anchor)
FROM aug;

INSERT WORK.NS_Spreads VALUES
       (N'ДЧБО', @Spread_DChbo),
       (N'Розничный бизнес', @Spread_Retail);

/* ── 3. Начальные циклы: размножение promo-окон ────────────── */
DROP TABLE IF EXISTS #cycles;
;WITH base AS (
    SELECT con_id, cli_id, out_rub, TSEGMENTNAME,
           win_start = dt_open,
           win_end   = DATEADD(day,-1,EOMONTH(DATEADD(month,1,dt_open))),
           cycle_id  = CAST(concat(con_id, '_0') AS nvarchar(50))
    FROM #bal_prom
)
, seq AS (
    SELECT * FROM base
    UNION ALL
    SELECT NULL, b.cli_id, b.out_rub, b.TSEGMENTNAME,
           win_start = DATEADD(day,1,b.win_end),
           win_end   = DATEADD(day,-1,EOMONTH(DATEADD(month,1,DATEADD(day,1,b.win_end)))),
           cycle_id  = CAST(concat(b.cli_id, '_', ROW_NUMBER() OVER(PARTITION BY b.cli_id ORDER BY b.win_end)+1) AS nvarchar(50))
    FROM seq b
    WHERE DATEADD(day,1,b.win_end) <= @HorizonTo
)
SELECT *
INTO   #cycles
FROM   seq
OPTION (MAXRECURSION 0);

/* ── 4. Денормализация: ставка на каждый день ──────────────── */
DROP TABLE IF EXISTS #raw;
;WITH calx AS (
    SELECT c.d, r.*
    FROM   #cal c
    JOIN   #cycles r ON c.d BETWEEN r.win_start AND DATEADD(day,1,r.win_end)
)
SELECT  r.cli_id, r.out_rub, r.TSEGMENTNAME, r.cycle_id,
        r.d AS dt_rep,
        rate_con =
            CASE 
              WHEN r.d < r.win_end THEN
                  (SELECT s.spread + k.KEY_RATE
                   FROM #key k JOIN WORK.NS_Spreads s ON s.TSEGMENTNAME = r.TSEGMENTNAME
                   WHERE k.DT_REP = r.win_start)
              WHEN r.d = r.win_end THEN
                  @BaseRate
              ELSE
                  (SELECT s.spread + k.KEY_RATE
                   FROM #key k JOIN WORK.NS_Spreads s ON s.TSEGMENTNAME = r.TSEGMENTNAME
                   WHERE k.DT_REP = r.d)
            END
INTO    #raw
FROM    calx r;

/* ── 5. Агрегация в финальную таблицу ──────────────────────── */
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;
SELECT dt_rep,
       SUM(out_rub) AS out_rub_total,
       SUM(out_rub * rate_con) / SUM(out_rub) AS rate_avg
INTO   WORK.Forecast_NS_Promo
FROM   #raw
GROUP BY dt_rep;

/* ── 6. PROMO-ставка на 1-е число каждого месяца ───────────── */
DROP TABLE IF EXISTS WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates(
  month_first date,
  TSEGMENTNAME nvarchar(40),
  promo_rate   decimal(9,4),
  PRIMARY KEY(month_first,TSEGMENTNAME));

;WITH mkey AS (
    SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) m1,
           MIN(KEY_RATE) key_min
    FROM   #key
    GROUP BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
INSERT WORK.NS_PromoRates
SELECT m.m1,
       s.TSEGMENTNAME,
       m.key_min + s.spread
FROM   mkey m
JOIN   WORK.NS_Spreads s ON 1=1;

/* ── 7. Просмотр результатов ───────────────────────────────── */
PRINT N'=== spread (Aug-25) ===';         SELECT * FROM WORK.NS_Spreads;
PRINT N'=== promo-rate (1-е) ===';        SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,TSEGMENTNAME;
PRINT N'=== promo-лента (TOP-40) ===';    SELECT TOP 40 * FROM WORK.Forecast_NS_Promo ORDER BY dt_rep;
GO
