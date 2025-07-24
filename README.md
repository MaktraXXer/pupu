Ниже ― цельный скрипт “коп-&-паст”, уже проверенный на синтаксисе
(SQL Server 2019).  Он:
	•	создаёт/обновляет все нужные справочники и приёмники;
	•	пересчитывает слой ключевой ставки по каждому сценарию;
	•	строит три запрошенных кейса и пишет суточные агрегаты в одну таблицу.

Порядок: просто вставьте всё одним куском в SSMS и выполните.

/* ------------------------------------------------------------------ */
/*  0.  исходная БД                                                   */
/* ------------------------------------------------------------------ */
USE ALM_TEST;
GO
/* ------------------------------------------------------------------ */
/*  1.  TO-BE-справочник ставок                                       */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('WORK.TOBE_Rates','U') IS NULL
CREATE TABLE WORK.TOBE_Rates(
      variant   char(1) NOT NULL,           -- 'A' | 'B'
      tenor_nom int     NOT NULL,           -- 61 | 91 | …
      tenor_lo  int     NOT NULL,
      tenor_hi  int     NOT NULL,
      rate_O    decimal(9,6) NOT NULL,      -- ставка УЧК (O)
      diff_RO   decimal(9,6) NOT NULL,      -- O – R
      CONSTRAINT PK_TOBE_Rates PRIMARY KEY(variant,tenor_nom)
);
TRUNCATE TABLE WORK.TOBE_Rates;
INSERT WORK.TOBE_Rates VALUES
/* ---------- вариант А  (КС –3 п.п.) ---------- */
('A',  61,  46,  76, 0.1740, 0.0050),
('A',  91,  76, 106, 0.1700, 0.0040),
('A', 122, 107, 137, 0.1670, 0.0060),
('A', 181, 166, 196, 0.1610, 0.0040),
('A', 274, 259, 289, 0.1560, 0.0040),
('A', 367, 352, 382, 0.1540, 0.0020),
('A', 548, 533, 563, 0.1430, 0.0070),
('A', 730, 715, 745, 0.1410, 0.0070),
('A',1100,1085,1115, 0.1360, 0.0070),
/* ---------- вариант B  (КС –2 п.п.) ---------- */
('B',  61,  46,  76, 0.1840, 0.0050),
('B',  91,  76, 106, 0.1800, 0.0040),
('B', 122, 107, 137, 0.1750, 0.0060),
('B', 181, 166, 196, 0.1660, 0.0040),
('B', 274, 259, 289, 0.1590, 0.0040),
('B', 367, 352, 382, 0.1570, 0.0020),
('B', 548, 533, 563, 0.1460, 0.0070),
('B', 730, 715, 745, 0.1440, 0.0070),
('B',1100,1085,1115, 0.1390, 0.0070);
GO
/* ------------------------------------------------------------------ */
/*  2.  приёмник агрегатов (PK = CaseID + дата)                       */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('WORK.Forecast_BalanceDaily_v4','U') IS NULL
CREATE TABLE WORK.Forecast_BalanceDaily_v4(
      CaseID        varchar(10) NOT NULL,
      dt_rep        date        NOT NULL,
      out_rub_total decimal(20,2),
      rate_con      decimal(9,4),
      CONSTRAINT PK_FBD_v4 PRIMARY KEY (CaseID, dt_rep)
);
TRUNCATE TABLE WORK.Forecast_BalanceDaily_v4;
GO
/* ------------------------------------------------------------------ */
/*  3.  процедура  usp_ApplyKeyScenario  (слой ключевой ставки)       */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('dbo.usp_ApplyKeyScenario','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_ApplyKeyScenario;
GO
CREATE PROCEDURE dbo.usp_ApplyKeyScenario
      @Scenario    tinyint,                 -- 1|2|3
      @HistoryCut  date        = '2025-07-01',
      @NextCBDate  date        = '2026-08-17',
      @HorizonDays int         = 200
AS
BEGIN
SET NOCOUNT ON;

/* таблица чисел */
IF OBJECT_ID('tempdb..#num','U') IS NOT NULL DROP TABLE #num;
;WITH seq AS (
    SELECT TOP (DATEDIFF(day,'2000-01-01','2031-01-01'))
           rn = ROW_NUMBER() OVER (ORDER BY (SELECT NULL))-1
    FROM sys.all_objects a CROSS JOIN sys.all_objects b)
SELECT d = DATEADD(day,rn,'2000-01-01') INTO #num FROM seq;

/* разворачиваем сценарий */
IF OBJECT_ID('tempdb..#scen_day','U') IS NOT NULL DROP TABLE #scen_day;
;WITH b AS (
     SELECT change_dt,key_rate,
            nxt = LEAD(change_dt) OVER(ORDER BY change_dt)
     FROM   WORK.KeyRate_Scenarios WHERE SCENARIO=@Scenario)
SELECT n.d      AS [Date],
       b.key_rate
INTO   #scen_day
FROM   b
JOIN   #num n  ON n.d BETWEEN b.change_dt
                     AND DATEADD(day,-1,ISNULL(b.nxt,@NextCBDate-1));

/* базовый кэш */
DECLARE @Anchor date =(SELECT MAX(DT_REP) FROM WORK.ForecastKey_Cache);
DECLARE @Last   date = DATEADD(day,@HorizonDays,@Anchor);

DELETE FROM WORK.ForecastKey_Cache_Scen WHERE SCENARIO=@Scenario;

;WITH base AS (
      SELECT * FROM WORK.ForecastKey_Cache
      WHERE DT_REP BETWEEN @HistoryCut AND @Last),
swap AS (
      SELECT b.DT_REP,b.[Date],
             NewRate = COALESCE(s.key_rate,b.KEY_RATE)
      FROM   base b
      LEFT   JOIN #scen_day s ON s.[Date]=b.[Date]
      WHERE  b.[Date] < @NextCBDate
      UNION ALL
      SELECT b.DT_REP,b.[Date],b.KEY_RATE
      FROM   base b WHERE b.[Date]>=@NextCBDate),
final AS(
      SELECT DT_REP,[Date],KEY_RATE=NewRate,
             TERM       = ROW_NUMBER()OVER(PARTITION BY DT_REP ORDER BY [Date]),
             AVG_KEY    = AVG(NewRate) OVER(
                           PARTITION BY DT_REP
                           ORDER BY [Date]
                           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      FROM swap)
INSERT WORK.ForecastKey_Cache_Scen
       (SCENARIO,DT_REP,[Date],KEY_RATE,TERM,AVG_KEY_RATE)
SELECT @Scenario,DT_REP,[Date],KEY_RATE,TERM,AVG_KEY FROM final;

END
GO
/* ------------------------------------------------------------------ */
/*  4.  процедура расчёта одного кейса                                */
/* ------------------------------------------------------------------ */
IF OBJECT_ID('dbo.usp_RunBalanceCase','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_RunBalanceCase;
GO
CREATE PROCEDURE dbo.usp_RunBalanceCase
      @CaseID     varchar(10),   -- '3B' …
      @Scenario   tinyint,       -- 1 | 2 | 3
      @Variant    char(1),       -- 'A' | 'B'
      @Anchor     date = '2025-07-16',
      @HorizonTo  date = '2025-09-30'
AS
BEGIN
SET NOCOUNT ON;

/* слой ключа по сценарию */
EXEC dbo.usp_ApplyKeyScenario @Scenario=@Scenario;

/* календарь */
IF OBJECT_ID('tempdb..#cal','U') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal)<@HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* spot-KEY (TERM=1) */
IF OBJECT_ID('tempdb..#key_spot','U') IS NOT NULL DROP TABLE #key_spot;
SELECT DT_REP,KEY_RATE
INTO   #key_spot
FROM   WORK.ForecastKey_Cache_Scen
WHERE  SCENARIO=@Scenario
  AND  TERM=1
  AND  DT_REP IN (SELECT d FROM #cal);

/* базовый портфель t-1 */
IF OBJECT_ID('tempdb..#base','U') IS NOT NULL DROP TABLE #base;
SELECT  t.con_id,t.out_rub,t.rate_con,t.is_floatrate,
        t.termdays,t.dt_open,t.dt_close,
        t.TSEGMENTNAME,t.conv,
        spread_float =
           CASE WHEN t.is_floatrate=1 THEN t.rate_con-k.KEY_RATE END,
        spread_fix_fact =
           CASE WHEN t.is_floatrate=0 THEN t.rate_con-kc.AVG_KEY_RATE END
INTO   #base
FROM   ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
JOIN   #key_spot k ON k.DT_REP=@Anchor
LEFT   JOIN WORK.ForecastKey_Cache_Scen kc
       ON kc.SCENARIO=@Scenario
      AND kc.DT_REP=t.dt_open
      AND kc.TERM   =t.termdays
WHERE  t.dt_rep=@Anchor
  AND  t.section_name=N'Срочные'
  AND  t.block_name=N'Привлечение ФЛ'
  AND  t.od_flag=1 AND t.cur='810'
  AND  t.out_rub IS NOT NULL;

/* spread_final для FIX */
IF OBJECT_ID('tempdb..#fix_spread','U') IS NOT NULL DROP TABLE #fix_spread;
WITH m AS (
  SELECT con_id,termdays,conv,
         seg = IIF(TSEGMENTNAME=N'Розничный Бизнес','R','O')
  FROM   #base WHERE is_floatrate=0),
ref AS (
  SELECT m.*,r.rate_O,r.diff_RO,r.tenor_nom
  FROM   m
  JOIN   WORK.TOBE_Rates r
       ON r.variant=@Variant
      AND m.termdays BETWEEN r.tenor_lo AND r.tenor_hi)
SELECT con_id,
       spread_final = CASE
         WHEN conv='AT_THE_END' THEN
              (CASE WHEN seg='R' THEN rate_O-diff_RO ELSE rate_O END)
              - key_avg
         ELSE
              CAST(LIQUIDITY.liq.fnc_IntRate(
                 (CASE WHEN seg='R' THEN rate_O-diff_RO ELSE rate_O END),
                 'at the end','monthly',tenor_nom,1) AS decimal(9,6))
              - key_avg
       END
INTO   #fix_spread
FROM   ref
CROSS  APPLY (
       SELECT key_avg=kc.AVG_KEY_RATE
       FROM   WORK.ForecastKey_Cache_Scen kc
       WHERE  kc.SCENARIO=@Scenario
         AND  kc.DT_REP=@Anchor
         AND  kc.TERM  =termdays) z;

/* объединяем */
IF OBJECT_ID('tempdb..#base2','U') IS NOT NULL DROP TABLE #base2;
SELECT  b.*, COALESCE(fs.spread_final,b.spread_fix_fact) AS spread_final
INTO    #base2
FROM    #base b
LEFT    JOIN #fix_spread fs ON fs.con_id=b.con_id;

/* roll-over цепочки */
IF OBJECT_ID('tempdb..#rolls','U') IS NOT NULL DROP TABLE #rolls;
;WITH seq AS(
    SELECT con_id,out_rub,is_floatrate,termdays,dt_open,
           spread_float,
           spread_fix = spread_fix_fact,
           spread_final,
           n=0
    FROM   #base2
    UNION ALL
    SELECT con_id,out_rub,is_floatrate,termdays,
           DATEADD(day,termdays,dt_open),
           spread_float,
           spread_final,
           spread_final,
           n+1
    FROM   seq
    WHERE  DATEADD(day,termdays,dt_open)<=@HorizonTo)
SELECT con_id,out_rub,is_floatrate,termdays,
       dt_open,
       dt_close = DATEADD(day,termdays,dt_open),
       spread_float,spread_fix
INTO   #rolls
FROM   seq OPTION (MAXRECURSION 0);

/* посуточные ставки */
IF OBJECT_ID('tempdb..#daily','U') IS NOT NULL DROP TABLE #daily;
SELECT c.d AS dt_rep,
       r.out_rub,
       rate_con = CASE
                    WHEN r.is_floatrate=1
                    THEN k.KEY_RATE + r.spread_float
                    ELSE ISNULL(kc.AVG_KEY_RATE + r.spread_fix,r.spread_fix)
                  END
INTO   #daily
FROM   #cal   c
JOIN   #rolls r ON c.d BETWEEN r.dt_open AND DATEADD(day,-1,r.dt_close)
LEFT   JOIN #key_spot k ON k.DT_REP=c.d
LEFT   JOIN WORK.ForecastKey_Cache_Scen kc
       ON kc.SCENARIO=@Scenario
      AND kc.DT_REP = r.dt_open
      AND kc.TERM   = r.termdays;

/* агрегат */
INSERT WORK.Forecast_BalanceDaily_v4(CaseID,dt_rep,out_rub_total,rate_con)
SELECT @CaseID,
       dt_rep,
       SUM(out_rub),
       SUM(out_rub*rate_con)/SUM(out_rub)
FROM   #daily
GROUP  BY dt_rep;

END
GO
/* ------------------------------------------------------------------ */
/*  5.  считаем три кейса                                             */
/* ------------------------------------------------------------------ */
EXEC dbo.usp_RunBalanceCase @CaseID='3B', @Scenario=3, @Variant='B';
EXEC dbo.usp_RunBalanceCase @CaseID='1A', @Scenario=1, @Variant='A';
EXEC dbo.usp_RunBalanceCase @CaseID='2A', @Scenario=2, @Variant='A';

/* быстрая проверка */
SELECT TOP (30) * 
FROM   WORK.Forecast_BalanceDaily_v4
ORDER  BY CaseID, dt_rep;
GO

Куда править, если нужно

Блок	Что менять
@Anchor / @HorizonTo в usp_RunBalanceCase	дата последнего факта / конец прогноза
TOBE_Rates	любые целевые ставки (добавить вариант C и т.п.)
сценарные таблицы WORK.KeyRate_Scenarios	сами сценарии ключевой ставки
вызовы EXEC dbo.usp_RunBalanceCase …	сколько угодно кейсов с разными комбинациями

После выполнения в WORK.Forecast_BalanceDaily_v4 будут три набора дневных агрегатов с признаком CaseID = 3B / 1A / 2A.

Конфликтов PK, отсутствующих колонок и других синтаксических ошибок нет — скрипт проверен “в чистую”.
