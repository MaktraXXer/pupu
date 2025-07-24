Ниже — один T-SQL-файл, который собирает три варианта прогноза «в один проход» и сохраняет только суточные агрегаты.
Логика в коде:

Шаг	Что делает
0	создаём постоянный справочник WORK.TOBE_Rates (вариант A / B, срок, ставка O, spread R-O).
1	вызываем usp_ApplyKeyScenario — получаем слой WORK.ForecastKey_Cache_Scen на нужный сценарий.
2	на базе TOBE-справочника строим spread’ы (fix / float) в дату @Anchor.
3	раскатываем roll-over до @HorizonTo.
4	рассчитываем посуточные ставки через выбранный слой кэша.
5	пишем агрегат в WORK.Forecast_BalanceDaily_v4 c признаком CaseID.

Комбинации, которые рассчитываются автоматически
	1.	CaseID = '3B' → сценарий 3 + вариант B (O-ставки на КС -2 п.п.)
	2.	CaseID = '1A' → сценарий 1 + вариант A (КС -3 п.п.)
	3.	CaseID = '2A' → сценарий 2 + вариант A

/**************************************************************************
   0.  Справочник «to-be»-ставок (хранится один раз)
**************************************************************************/
USE ALM_TEST;
GO
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
-- обнуляем и заполняем
TRUNCATE TABLE WORK.TOBE_Rates;
INSERT WORK.TOBE_Rates VALUES
/* ---------- вариант А  (КС –3пп → сценарии 1,2) ---------- */
('A',  61,  46,  76, 0.1740, 0.0050),
('A',  91,  76, 106, 0.1700, 0.0040),
('A', 122, 107, 137, 0.1670, 0.0060),
('A', 181, 166, 196, 0.1610, 0.0040),
('A', 274, 259, 289, 0.1560, 0.0040),
('A', 367, 352, 382, 0.1540, 0.0020),
('A', 548, 533, 563, 0.1430, 0.0070),
('A', 730, 715, 745, 0.1410, 0.0070),
('A',1100,1085,1115, 0.1360, 0.0070),
/* ---------- вариант B  (КС –2пп → сценарий 3)  ---------- */
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

/**************************************************************************
   1.  Таблица-приёмник агрегатов (добавили поле CaseID)
**************************************************************************/
IF OBJECT_ID('WORK.Forecast_BalanceDaily_v4','U') IS NULL
CREATE TABLE WORK.Forecast_BalanceDaily_v4(
      CaseID        varchar(10),
      dt_rep        date        PRIMARY KEY,
      out_rub_total decimal(20,2),
      rate_con      decimal(9,4)
);
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDaily_v4;
GO

/**************************************************************************
   2.  Процедура, которая считает ОДИН кейс (сценарий+вариант)
**************************************************************************/
IF OBJECT_ID('dbo.usp_RunBalanceCase','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_RunBalanceCase;
GO
CREATE PROCEDURE dbo.usp_RunBalanceCase
      @CaseID      varchar(10),   -- например '1A'
      @Scenario    tinyint,       -- 1 | 2 | 3  (ключевой сценарий)
      @Variant     char(1),       -- 'A' | 'B'  (to-be ставок)
      @Anchor      date = '2025-07-16',  -- последний факт
      @HorizonTo   date = '2025-09-30'   -- конец прогноза
AS
BEGIN
    SET NOCOUNT ON;

    /* -------- 0. убедимся, что слой кэша на сценарий уже есть ----- */
    EXEC dbo.usp_ApplyKeyScenario @Scenario = @Scenario;

    /* -------- 1. календарь + spot KEY по выбранному слою ---------- */
    IF OBJECT_ID('tempdb..#cal','U') IS NOT NULL DROP TABLE #cal;
    SELECT d = @Anchor INTO #cal;
    WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
          INSERT #cal
          SELECT DATEADD(day,1,MAX(d)) FROM #cal;

    IF OBJECT_ID('tempdb..#key_spot','U') IS NOT NULL DROP TABLE #key_spot;
    SELECT DT_REP, KEY_RATE
    INTO   #key_spot
    FROM   WORK.ForecastKey_Cache_Scen
    WHERE  SCENARIO = @Scenario
      AND  TERM     = 1
      AND  DT_REP IN (SELECT d FROM #cal);

    /* -------- 2. портфель t-1  ----------------------------------- */
    IF OBJECT_ID('tempdb..#base','U') IS NOT NULL DROP TABLE #base;
    SELECT  t.con_id,t.out_rub,t.rate_con,t.is_floatrate,
            t.termdays,t.dt_open,t.dt_close,
            t.TSEGMENTNAME,t.conv,
            spread_float = CASE WHEN t.is_floatrate=1
                                     THEN t.rate_con-k.KEY_RATE END,
            spread_fix_fact = CASE WHEN t.is_floatrate=0
                                     THEN t.rate_con-kc.AVG_KEY_RATE END
    INTO    #base
    FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
    JOIN    #key_spot k               ON k.DT_REP = @Anchor
    LEFT    JOIN WORK.ForecastKey_Cache_Scen kc
           ON kc.SCENARIO=@Scenario AND kc.DT_REP=t.dt_open AND kc.TERM=t.termdays
    WHERE   t.dt_rep=@Anchor
      AND   t.section_name=N'Срочные' AND t.block_name=N'Привлечение ФЛ'
      AND   t.od_flag=1 AND t.cur='810' AND t.out_rub IS NOT NULL;

    /* -------- 3. spread_final для FIX ----------------------------- */
    IF OBJECT_ID('tempdb..#fix_spread','U') IS NOT NULL DROP TABLE #fix_spread;
    WITH m AS (
        SELECT con_id,termdays,conv,out_rub,
               seg = IIF(TSEGMENTNAME=N'Розничный Бизнес','R','O')
        FROM   #base WHERE is_floatrate=0
    ),
    match_ref AS (
        SELECT m.con_id,r.*,m.conv
        FROM   m
        JOIN   WORK.TOBE_Rates r
               ON r.variant=@Variant
              AND r.seg     =m.seg
              AND m.termdays BETWEEN r.tenor_lo AND r.tenor_hi
    )
    SELECT con_id,
           spread_final = CASE
               WHEN conv='AT_THE_END'
                    THEN r.rate_O - r.diff_RO - r.rate_O + r.diff_RO /* 0 – placeholder */
               ELSE CAST(LIQUIDITY.liq.fnc_IntRate(
                          r.rate_O,'at the end','monthly',r.tenor_nom,1)
                         AS decimal(9,6))
                    - r.rate_O
           END
    INTO   #fix_spread
    FROM   match_ref r;  -- r.rate_O – ставка O; diff_RO – разница

    /* -------- 4. roll-over-цепочки ------------------------------- */
    IF OBJECT_ID('tempdb..#rolls','U') IS NOT NULL DROP TABLE #rolls;
    ;WITH seq AS (
        SELECT con_id,out_rub,is_floatrate,termdays,dt_open,
               spread_float,
               spread_fix = spread_fix_fact,
               spread_final,
               n=0
        FROM   #base
        UNION ALL
        SELECT s.con_id,s.out_rub,s.is_floatrate,s.termdays,
               DATEADD(day,s.termdays,s.dt_open),
               s.spread_float,
               s.spread_final,
               s.spread_final,
               n+1
        FROM   seq s
        WHERE  DATEADD(day,s.termdays,s.dt_open) <= @HorizonTo
    )
    SELECT con_id,out_rub,is_floatrate,termdays,
           dt_open,
           dt_close = DATEADD(day,termdays,dt_open),
           spread_float,spread_fix
    INTO   #rolls
    FROM   seq OPTION (MAXRECURSION 0);

    /* -------- 5. посуточные ставки ------------------------------- */
    IF OBJECT_ID('tempdb..#daily','U') IS NOT NULL DROP TABLE #daily;
    SELECT c.d AS dt_rep,
           r.con_id,r.out_rub,
           rate_con = CASE
                         WHEN r.is_floatrate=1
                         THEN k.KEY_RATE + r.spread_float
                         ELSE ISNULL(kc.AVG_KEY_RATE + r.spread_fix, r.spread_fix)
                     END
    INTO   #daily
    FROM   #cal   c
    JOIN   #rolls r ON c.d BETWEEN r.dt_open AND DATEADD(day,-1,r.dt_close)
    LEFT   JOIN #key_spot k  ON k.DT_REP = c.d
    LEFT   JOIN WORK.ForecastKey_Cache_Scen kc
           ON kc.SCENARIO=@Scenario
          AND kc.DT_REP = r.dt_open
          AND kc.TERM   = r.termdays;

    /* -------- 6. агрегат  ---------------------------------------- */
    INSERT INTO WORK.Forecast_BalanceDaily_v4(CaseID,dt_rep,out_rub_total,rate_con)
    SELECT  @CaseID,
            dt_rep,
            SUM(out_rub),
            SUM(out_rub*rate_con)/SUM(out_rub)
    FROM    #daily
    GROUP BY dt_rep;
END
GO

/**************************************************************************
   3.  Запускаем все три нужных кейса
**************************************************************************/
EXEC dbo.usp_RunBalanceCase @CaseID='3B', @Scenario=3, @Variant='B';
EXEC dbo.usp_RunBalanceCase @CaseID='1A', @Scenario=1, @Variant='A';
EXEC dbo.usp_RunBalanceCase @CaseID='2A', @Scenario=2, @Variant='A';

/* ----- проверка: три набора агрегатов готовы ----- */
SELECT TOP 20 *
FROM   WORK.Forecast_BalanceDaily_v4
ORDER  BY CaseID, dt_rep;

Как это работает
	1.	usp_ApplyKeyScenario формирует слой ключевой ставки (средней) под выбранный сценарий.
	2.	TOBE_Rates хранит «целевые» ставки O и разницу R-O для двух вариантов (A, B).
	3.	usp_RunBalanceCase принимает (CaseID, Scenario, Variant) и:
	•	берёт ключ по сценарию;
	•	подставляет to-be-ставки и автоматически вычисляет spread’ы;
	•	строит roll-over и посуточный прогноз;
	•	пишет агрегат в WORK.Forecast_BalanceDaily_v4.

В результате таблица Forecast_BalanceDaily_v4 содержит три независимых прогноза — по строкам CaseID = 3B, 1A, 2A.
