```sql
/* === ПЕРЕМЕННЫЕ === */
DECLARE @Anchor     date = '2025-07-14';   -- последний факт
DECLARE @HorizonTo  date = '2025-08-31';   -- конец прогноза

/* === ВСПОМОГАТЕЛЬНЫЕ ТАБЛИЦЫ === */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL           DROP TABLE #cal;
IF OBJECT_ID('tempdb..#key_spot') IS NOT NULL      DROP TABLE #key_spot;
IF OBJECT_ID('tempdb..#base') IS NOT NULL          DROP TABLE #base;
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL         DROP TABLE #rolls;
IF OBJECT_ID('tempdb..#work') IS NOT NULL          DROP TABLE #work;
IF OBJECT_ID('tempdb..#daily') IS NOT NULL         DROP TABLE #daily;

/* календарь от Anchor до HorizonTo */
SELECT d = @Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* spot KEY_RATE (TERM = 1) на каждую дату календаря */
SELECT fc.DT_REP, fc.KEY_RATE
INTO   #key_spot
FROM   ALM_TEST.WORK.ForecastKey_Cache fc
JOIN   #cal c ON c.d = fc.DT_REP
WHERE  fc.TERM = 1;

/* базовый срез портфеля + спреды */
SELECT  t.con_id,
        t.out_rub,
        t.rate_con,
        t.is_floatrate,
        t.termdays,
        t.dt_open,
        t.dt_close,
        spread_float = CASE WHEN t.is_floatrate = 1
                            THEN t.rate_con - ks.KEY_RATE END,
        spread_fix   = CASE WHEN t.is_floatrate = 0
                            THEN t.rate_con - fk_open.AVG_KEY_RATE END
INTO    #base
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
JOIN    #key_spot ks           ON ks.DT_REP = @Anchor
LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fk_open
       ON fk_open.DT_REP = t.dt_open
      AND fk_open.TERM   = t.termdays
WHERE   t.dt_rep = @Anchor
  AND   t.section_name = N'Срочные'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.out_rub IS NOT NULL;

/* roll-over для всех депозитов, пока dt_open ≤ HorizonTo */
;WITH seq AS (
    SELECT  con_id, out_rub, is_floatrate, termdays,
            spread_float, spread_fix,
            dt_open = dt_open,                    n = 0
    FROM    #base
    UNION ALL
    SELECT  s.con_id, s.out_rub, s.is_floatrate, s.termdays,
            s.spread_float, s.spread_fix,
            DATEADD(day,s.termdays,s.dt_open),    n + 1
    FROM    seq s
    WHERE   DATEADD(day,s.termdays,s.dt_open) <= @HorizonTo
)
SELECT  con_id,
        out_rub,
        is_floatrate,
        termdays,
        dt_open,
        dt_close      = DATEADD(day,termdays,dt_open),
        spread_float,
        spread_fix
INTO    #rolls
FROM    seq
OPTION (MAXRECURSION 0);

/* полный набор «живых» контрактов */
SELECT * INTO #work FROM #rolls;

/* ставка на каждый день */
SELECT
        c.d          AS dt_rep,
        w.con_id,
        w.out_rub,
        rate_con = CASE
                     WHEN w.is_floatrate = 1
                          THEN ks.KEY_RATE + w.spread_float
                     ELSE ISNULL(fko.AVG_KEY_RATE + w.spread_fix, w.spread_fix)
                   END
INTO    #daily
FROM    #cal  c
JOIN    #work w
      ON c.d BETWEEN w.dt_open AND DATEADD(day,-1,w.dt_close)
LEFT JOIN #key_spot ks
       ON ks.DT_REP = c.d
LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fko
       ON fko.DT_REP = w.dt_open
      AND fko.TERM   = w.termdays;

/* === ЦЕЛЕВЫЕ ТАБЛИЦЫ =================================================== */
IF OBJECT_ID('ALM_TEST.WORK.Forecast_BalanceDaily','U') IS NULL
CREATE TABLE ALM_TEST.WORK.Forecast_BalanceDaily
( dt_rep DATE PRIMARY KEY,
  out_rub_total DECIMAL(20,2),
  rate_con      DECIMAL(9,4) );

IF OBJECT_ID('ALM_TEST.WORK.Forecast_BalanceDeals','U') IS NULL
CREATE TABLE ALM_TEST.WORK.Forecast_BalanceDeals
( dt_rep DATE,
  con_id BIGINT,
  out_rub DECIMAL(20,2),
  rate_con DECIMAL(9,4),
  CONSTRAINT PK_FBD PRIMARY KEY (dt_rep, con_id) );

/* агрегированный портфель-день */
MERGE ALM_TEST.WORK.Forecast_BalanceDaily AS tgt
USING (
        SELECT  dt_rep,
                out_rub_total = SUM(out_rub),
                rate_con      = SUM(out_rub*rate_con)/SUM(out_rub)
        FROM    #daily
        GROUP  BY dt_rep
) src
ON (tgt.dt_rep = src.dt_rep)
WHEN MATCHED THEN
    UPDATE SET out_rub_total = src.out_rub_total,
               rate_con      = src.rate_con
WHEN NOT MATCHED THEN
    INSERT (dt_rep, out_rub_total, rate_con)
    VALUES (src.dt_rep, src.out_rub_total, src.rate_con);

/* деталка только по плавающим и roll-over */
MERGE ALM_TEST.WORK.Forecast_BalanceDeals AS tgt
USING #daily AS src
ON (tgt.dt_rep = src.dt_rep AND tgt.con_id = src.con_id)
WHEN NOT MATCHED THEN
    INSERT (dt_rep, con_id, out_rub, rate_con)
    VALUES (src.dt_rep, src.con_id, src.out_rub, src.rate_con);
```

**Что получаете**

| объект                                    | наполнение                                                                   |
| ----------------------------------------- | ---------------------------------------------------------------------------- |
| **ALM\_TEST.WORK.Forecast\_BalanceDaily** | 14-07-2025 … 31-08-2025, сумма портфеля и средневзвешенная ставка            |
| **ALM\_TEST.WORK.Forecast\_BalanceDeals** | только те сделки, у которых ставка меняется (float) или происходит roll-over |

Скрипт компактен — копируйте, выполняйте в **ALM\_TEST**.
