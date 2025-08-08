/* ═══════════════════════════════════════════════════════════════
   NS-forecast — ЧАСТЬ 1  (ТОЛЬКО FLOAT и FIX-base)
   СНАПШОТ @Anchor = как в mail.usp_fill_balance_metrics_savings
   v.2025-08-10 (без IF)
════════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31';

/* 1) календарь */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* 2) KEY-spot (TERM=1) */
DROP TABLE IF EXISTS #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* 3) СНАПШОТ @Anchor как в mail.usp_fill_balance_metrics_savings */
DROP TABLE IF EXISTS #bal_src;
SELECT
    t.dt_rep,
    CAST(t.dt_open  AS date)                AS dt_open,
    CAST(t.dt_close AS date)                AS dt_close,
    t.con_id,
    t.cli_id,
    t.prod_id,
    CAST(t.out_rub AS decimal(20,2))        AS out_rub,
    CAST(t.rate_con AS decimal(9,4))        AS rate_balance,
    t.rate_con_src,
    t.is_floatrate,
    t.TSEGMENTNAME,
    r.rate                                   AS rate_liq
INTO #bal_src
FROM   alm.ALM.vw_balance_rest_all AS t WITH (NOLOCK)
LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = t.con_id
       AND CASE WHEN t.dt_open = t.dt_rep
                THEN DATEADD(day,1,t.dt_rep)
                ELSE t.dt_rep
           END BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day, 2,@Anchor)
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.cur          = '810';

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src (con_id, dt_rep);

;WITH bal_pos AS (
    SELECT *,
           MIN(CASE WHEN rate_balance > 0
                     AND rate_con_src = N'счет ультра,вручную'
                    THEN rate_balance END)
               OVER (PARTITION BY con_id
                     ORDER BY dt_rep
                     ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
    FROM #bal_src
    WHERE dt_rep = @Anchor
),
rate_calc AS (
    SELECT *,
           CASE
             WHEN rate_liq IS NULL
                  THEN CASE
                           WHEN rate_balance < 0
                                THEN COALESCE(rate_pos, rate_balance)
                           ELSE rate_balance
                       END
             WHEN rate_liq < 0  AND rate_balance > 0 THEN rate_balance
             WHEN rate_liq < 0  AND rate_balance < 0 THEN COALESCE(rate_pos, rate_balance)
             WHEN rate_liq >= 0 AND rate_balance >= 0 THEN rate_liq
             WHEN rate_liq > 0  AND rate_balance < 0 THEN rate_liq
             ELSE rate_liq
           END AS rate_use
    FROM bal_pos
)
DROP TABLE IF EXISTS #bal;
SELECT
    con_id, cli_id, prod_id,
    out_rub,
    rate_con = CAST(rate_use AS decimal(9,4)),
    dt_open, dt_close,
    is_floatrate,
    TSEGMENTNAME
INTO #bal
FROM rate_calc
WHERE out_rub IS NOT NULL;

/* 4-A) FLOAT daily — спред фиксируем на Anchor */
DROP TABLE IF EXISTS #FLOAT_daily;
SELECT  b.con_id,
        b.cli_id,
        b.TSEGMENTNAME,
        b.out_rub,
        c.d                                      AS dt_rep,
        (b.rate_con - k0.KEY_RATE) + k1.KEY_RATE AS rate_con
INTO    #FLOAT_daily
FROM    #bal b
JOIN    #key k0 ON k0.DT_REP = @Anchor
JOIN    #cal c  ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
JOIN    #key k1 ON k1.DT_REP = c.d
WHERE   b.prod_id = 3103;

/* агрегат FLOAT — дроп и create заново */
DROP TABLE IF EXISTS WORK.Forecast_NS_Float;
CREATE TABLE WORK.Forecast_NS_Float(
    dt_rep date PRIMARY KEY,
    out_rub_total  decimal(20,2),
    rate_avg       decimal(9,4)
);

INSERT WORK.FOrecast_NS_Float
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #FLOAT_daily
GROUP  BY dt_rep;

/* 4-B) FIX-base daily (prod 654, dt_open < 2025-07-01) */
DROP TABLE IF EXISTS #FIX_base_daily;
SELECT  b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
        c.d            AS dt_rep,
        b.rate_con     AS rate_con
INTO    #FIX_base_daily
FROM    #bal b
JOIN    #cal c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE   b.prod_id = 654
  AND   b.dt_open < '2025-07-01';

/* агрегат FIX-base — дроп и create заново */
DROP TABLE IF EXISTS WORK.Forecast_NS_FixBase;
CREATE TABLE WORK.Forecast_NS_FixBase(
    dt_rep date PRIMARY KEY,
    out_rub_total  decimal(20,2),
    rate_avg       decimal(9,4)
);

INSERT WORK.Forecast_NS_FixBase
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #FIX_base_daily
GROUP  BY dt_rep;

/* 5) проверки */
PRINT N'==== FLOAT daily (TOP-30) ====';
SELECT TOP (30) * FROM #FLOAT_daily ORDER BY dt_rep,con_id;

PRINT N'==== FIX-base daily (TOP-30) ====';
SELECT TOP (30) * FROM #FIX_base_daily ORDER BY dt_rep,con_id;

PRINT N'==== агрегаты (первые 10 дней) ====';
SELECT 'FLOAT' AS bucket,* FROM WORK.Forecast_NS_Float   ORDER BY dt_rep;
SELECT 'FIX-base',*       FROM WORK.Forecast_NS_FixBase  ORDER BY dt_rep;
GO
