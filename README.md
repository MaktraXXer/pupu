USE ALM_TEST;
GO
IF OBJECT_ID('dbo.usp_ApplyKeyScenario','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_ApplyKeyScenario;
GO
CREATE PROCEDURE dbo.usp_ApplyKeyScenario
      @Scenario     tinyint = 1,                -- 1 | 2 …
      @HistoryCut   date    = '2025-07-01',    -- с этой даты правим
      @NextCBDate   date    = '2026-08-17',    -- след. заседание (факт)
      @HorizonDays  int     = 200              -- сколько dt_rep вперёд
AS
BEGIN
    SET NOCOUNT ON;

    /* 0. таблица-чисел: одна строка = один день --------------------*/
    IF OBJECT_ID('tempdb..#num','U') IS NOT NULL DROP TABLE #num;
    ;WITH seq AS (
        SELECT TOP (DATEDIFF(day,'2000-01-01','2031-01-01'))
               rn = ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1
        FROM sys.all_objects a CROSS JOIN sys.all_objects b
    )
    SELECT  d = DATEADD(day, seq.rn, '2000-01-01')
    INTO    #num
    FROM    seq;                                  -- ← rn внутри CTE, здесь не нужен

    /* 1. сценарий «день-в-день» ------------------------------------*/
    IF OBJECT_ID('tempdb..#scen_day','U') IS NOT NULL DROP TABLE #scen_day;
    ;WITH bnd AS (
        SELECT  change_dt,
                key_rate,
                nxt = LEAD(change_dt) OVER (ORDER BY change_dt)
        FROM    WORK.KeyRate_Scenarios
        WHERE   SCENARIO = @Scenario
    )
    SELECT  n.d            AS [Date],
            b.key_rate
    INTO    #scen_day
    FROM    bnd b
    JOIN    #num n
           ON n.d BETWEEN b.change_dt
                      AND DATEADD(day,-1, ISNULL(b.nxt, DATEADD(day,-1,@NextCBDate)));

    /* 2. диапазон базового кэша -----------------------------------*/
    DECLARE @AnchorFact date = (SELECT MAX(DT_REP) FROM WORK.ForecastKey_Cache); -- t-1
    DECLARE @LastSnap  date = DATEADD(day, @HorizonDays, @AnchorFact);

    /* 3. формируем слой сценария ----------------------------------*/
    DELETE FROM WORK.ForecastKey_Cache_Scen WHERE SCENARIO = @Scenario;

    ;WITH base AS (
        SELECT * FROM WORK.ForecastKey_Cache
        WHERE  DT_REP BETWEEN @HistoryCut AND @LastSnap
    ),
    merged AS (
        /* 3-a. до @NextCBDate подменяем ставкой сценария */
        SELECT  b.DT_REP, b.[Date],
                NewRate = COALESCE(s.key_rate, b.KEY_RATE)
        FROM    base b
        LEFT    JOIN #scen_day s ON s.[Date] = b.[Date]
        WHERE   b.[Date] < @NextCBDate
        UNION ALL
        /* 3-b. после @NextCBDate оставляем факт/старый прогноз */
        SELECT  b.DT_REP, b.[Date], b.KEY_RATE
        FROM    base b
        WHERE   b.[Date] >= @NextCBDate
    ),
    final AS (
        SELECT  m.DT_REP,
                m.[Date],
                KEY_RATE     = m.NewRate,
                TERM         = ROW_NUMBER() OVER
                                 (PARTITION BY m.DT_REP ORDER BY m.[Date]),
                AVG_KEY_RATE = AVG(m.NewRate)  OVER
                                 (PARTITION BY m.DT_REP
                                  ORDER BY     m.[Date]
                                  ROWS BETWEEN UNBOUNDED PRECEDING
                                       AND   CURRENT ROW)
        FROM    merged m
    )
    /* 3-c. пишем результат */
    INSERT INTO WORK.ForecastKey_Cache_Scen
           (SCENARIO, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE)
    SELECT  @Scenario, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
    FROM    final;

    /* 4. «доправочная» история (DT_REP < @HistoryCut) -------------*/
    INSERT INTO WORK.ForecastKey_Cache_Scen
           (SCENARIO, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE)
    SELECT  @Scenario, DT_REP, [Date], KEY_RATE, TERM, AVG_KEY_RATE
    FROM    WORK.ForecastKey_Cache
    WHERE   DT_REP < @HistoryCut;
END
GO
