/* ═══════════════════════════════════════════════════════════════
   NS-forecast — ЧАСТЬ 1  (ТОЛЬКО FLOAT и FIX-base)
   ⚠ СНАПШОТ @Anchor = как в mail.usp_fill_balance_metrics_savings:
     rate_use по ULTRA (+1/+2), LIQ vs balance, и ТЕ ЖЕ фильтры.
   * FLOAT     → prod_id 3103, спред фиксируем на Anchor
   * FIX-base  → prod_id 654, dt_open < 2025-07-01, ставка константа = rate_use(@Anchor)
   создаёт/перезаписывает:
     #cal, #key, #bal_src, #bal
     #FLOAT_daily            + WORK.Forecast_NS_Float
     #FIX_base_daily         + WORK.Forecast_NS_FixBase
   v.2025-08-10
════════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO

DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31';

/* ───────────── 1) Календарь ───────────── */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d = @Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* ───────────── 2) KEY-spot (TERM=1) ───── */
IF OBJECT_ID('tempdb..#key') IS NOT NULL DROP TABLE #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* ───────────── 3) СНАПШОТ @Anchor как в SP mail.usp_fill_balance_metrics_savings ─────────────
   Берём окно @Anchor±2 для look-ahead, тянем LIQ ставки с ULTRA-сдвигом, считаем rate_use и
   оставляем строки РОВНО на @Anchor.
*/
IF OBJECT_ID('tempdb..#bal_src') IS NOT NULL DROP TABLE #bal_src;
SELECT
    t.dt_rep,
    CAST(t.dt_open  AS date)                 AS dt_open,
    CAST(t.dt_close AS date)                 AS dt_close,
    CAST(t.con_id   AS bigint)               AS con_id,
    CAST(t.cli_id   AS bigint)               AS cli_id,
    CAST(t.prod_id  AS int)                  AS prod_id,
    CAST(t.out_rub  AS decimal(20,2))        AS out_rub,
    CAST(t.rate_con AS decimal(9,4))         AS rate_balance,   -- ставка из баланса (t)
    t.rate_con_src,
    t.TSEGMENTNAME,
    CAST(r.rate    AS decimal(9,4))          AS rate_liq
INTO #bal_src
FROM   alm.ALM.vw_balance_rest_all AS t WITH (NOLOCK)
LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = t.con_id
       AND CASE WHEN t.dt_open = t.dt_rep
                THEN DATEADD(day,1,t.dt_rep)          -- первый «нулевой» день ULTRA → +1
                ELSE t.dt_rep
           END BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day,2,@Anchor)
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.cur          = '810';

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src (con_id, dt_rep);

IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;

;WITH bal_pos AS (
    SELECT *,
           /* первая положительная ставка ULTRA на +1 / +2 день */
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
             /* r IS NULL */
             WHEN rate_liq IS NULL
                  THEN CASE
                           WHEN rate_balance < 0
                                THEN COALESCE(rate_pos, rate_balance)
                           ELSE rate_balance
                       END
             /* (1) r<0 , t>0 */
             WHEN rate_liq < 0  AND rate_balance > 0 THEN rate_balance
             /* (2) r<0 , t<0 */
             WHEN rate_liq < 0  AND rate_balance < 0 THEN COALESCE(rate_pos, rate_balance)
             /* (3) r≥0 , t≥0 */
             WHEN rate_liq >= 0 AND rate_balance >= 0 THEN rate_liq
             /* (4) r>0 , t<0 */
             WHEN rate_liq > 0  AND rate_balance < 0 THEN rate_liq
             ELSE rate_liq
           END AS rate_use
    FROM bal_pos
)
SELECT
    con_id,
    cli_id,
    prod_id,
    out_rub,
    rate_con = CAST(rate_use AS decimal(9,4)),   -- унифицируем имя
    dt_open,
    dt_close,
    TSEGMENTNAME
INTO #bal
FROM rate_calc
WHERE out_rub IS NOT NULL;

/* ───────────── 4-A) FLOAT daily (спред фиксируем на Anchor) ─────────────
   spread = rate_con(@Anchor) - KEY(@Anchor); на каждый день d → rate = spread + KEY(d)
*/
IF OBJECT_ID('tempdb..#FLOAT_daily') IS NOT NULL DROP TABLE #FLOAT_daily;

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

/* агрегат FLOAT */
IF OBJECT_ID('WORK.Forecast_NS_Float','U') IS NOT NULL
    TRUNCATE TABLE WORK.Forecast_NS_Float;
ELSE
    CREATE TABLE WORK.Forecast_NS_Float(
        dt_rep date PRIMARY KEY,
        out_rub_total  decimal(20,2),
        rate_avg       decimal(9,4)
    );

INSERT WORK.Forecast_NS_Float (dt_rep,out_rub_total,rate_avg)
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #FLOAT_daily
GROUP  BY dt_rep;

/* ───────────── 4-B) FIX-base daily (prod 654, dt_open < 2025-07-01) ─────────────
   ставка = rate_use(@Anchor) константой по горизонту
*/
IF OBJECT_ID('tempdb..#FIX_base_daily') IS NOT NULL DROP TABLE #FIX_base_daily;

SELECT  b.con_id,
        b.cli_id,
        b.TSEGMENTNAME,
        b.out_rub,
        c.d            AS dt_rep,
        b.rate_con     AS rate_con
INTO    #FIX_base_daily
FROM    #bal b
JOIN    #cal c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE   b.prod_id = 654
  AND   b.dt_open < '2025-07-01';

/* агрегат FIX-base */
IF OBJECT_ID('WORK.Forecast_NS_FixBase','U') IS NOT NULL
    TRUNCATE TABLE WORK.Forecast_NS_FixBase;
ELSE
    CREATE TABLE WORK.Forecast_NS_FixBase(
        dt_rep date PRIMARY KEY,
        out_rub_total  decimal(20,2),
        rate_avg       decimal(9,4)
    );

INSERT WORK.Forecast_NS_FixBase (dt_rep,out_rub_total,rate_avg)
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #FIX_base_daily
GROUP  BY dt_rep;

/* ───────────── 5) контрольный вывод ───────────── */
PRINT N'==== FLOAT daily (TOP-30) ====';
SELECT TOP (30) * FROM #FLOAT_daily ORDER BY dt_rep, con_id;

PRINT N'==== FIX-base daily (TOP-30) ====';
SELECT TOP (30) * FROM #FIX_base_daily ORDER BY dt_rep, con_id;

PRINT N'==== агрегаты (первые 10 дней) ====';
SELECT 'FLOAT'   AS bucket, * FROM WORK.Forecast_NS_Float   ORDER BY dt_rep;
SELECT 'FIX-base'          , * FROM WORK.Forecast_NS_FixBase ORDER BY dt_rep;
GO
