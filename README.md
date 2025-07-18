/**************************************************************************
   V-3 FIN  (18 июл 2025)
   – ручной «TO-BE» справочник (±15 дн);
   – 31 ± 10 дн переводим в «91»;
   – для fix-депо берём: spread_final  ⟵  TO-BE | факт;
   – float-депо не трогаем.
**************************************************************************/
USE ALM_TEST;
GO
/*────────────  ПАРАМЕТРЫ  ────────────*/
DECLARE
    @Anchor    date = '2025-07-16',           -- последний факт
    @HorizonTo date = '2025-09-30';           -- крайняя дата прогноза

/*=======================================================================
  0. СПРАВОЧНИК  TO-BE / KEY  (±15 дн)
=======================================================================*/
IF OBJECT_ID('tempdb..#ref_spread') IS NOT NULL DROP TABLE #ref_spread;
CREATE TABLE #ref_spread
( seg char(1),        -- R = Розничный бизнес ; O = прочие
  term_nom int,       -- «опорный» срок
  term_lo  int,       -- окно −15
  term_hi  int,       -- окно +15
  tobe_end decimal(9,6),   -- ставка (конец срока)
  key_avg  decimal(9,6) ); -- средний прогнозный KEY

INSERT #ref_spread VALUES
/* … R … */ ('R',  61,  46,  76, 0.1820, 0.174918),
             ('R',  91,  76, 106, 0.1790, 0.173098),
             ('R', 122, 107, 137, 0.1810, 0.169306),
             ('R', 181, 166, 196, 0.1750, 0.163700),
             ('R', 274, 259, 289, 0.1610, 0.156300),
             ('R', 367, 352, 382, 0.1610, 0.150200),
             ('R', 550, 535, 565, 0.1430, 0.142100),
             ('R', 730, 715, 745, 0.1410, 0.137800),
             ('R',1100,1085,1115, 0.1360, 0.133200),
/* … O … */ ('O',  61,  46,  76, 0.1860, 0.174918),
             ('O',  91,  76, 106, 0.1830, 0.173098),
             ('O', 122, 107, 137, 0.1850, 0.169306),
             ('O', 181, 166, 196, 0.1790, 0.163700),
             ('O', 274, 259, 289, 0.1660, 0.156300),
             ('O', 367, 352, 382, 0.1660, 0.150200),
             ('O', 550, 535, 565, 0.1500, 0.142100),
             ('O', 730, 715, 745, 0.1480, 0.137800),
             ('O',1100,1085,1115, 0.1430, 0.133200);

/*=======================================================================
  1. FACT-портфель на 16-07-2025  (всё нужное сразу)
=======================================================================*/
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;

/* календарь spot KEY на датах расчёта */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d = @Anchor
INTO   #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

IF OBJECT_ID('tempdb..#key_spot') IS NOT NULL DROP TABLE #key_spot;
SELECT fc.DT_REP, fc.KEY_RATE
INTO   #key_spot
FROM   WORK.ForecastKey_Cache fc
JOIN   #cal c ON c.d = fc.DT_REP
WHERE  fc.TERM = 1;

/* сам портфель */
SELECT  t.con_id,
        t.out_rub,
        t.rate_con,
        t.is_floatrate,
        t.termdays,
        t.dt_open,
        t.dt_close,
        t.TSEGMENTNAME,
        t.conv,
        /* спред-float = факт */
        spread_float = CASE WHEN t.is_floatrate = 1
                                THEN t.rate_con - ks.KEY_RATE END,
        /* спред-fix: пока факт; позже заменим через #fix_spread */
        spread_fix_fact = CASE WHEN t.is_floatrate = 0
                                    THEN t.rate_con - fk_open.AVG_KEY_RATE END
INTO    #base
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
JOIN    #key_spot ks
           ON ks.DT_REP = @Anchor
LEFT    JOIN WORK.ForecastKey_Cache fk_open
           ON fk_open.DT_REP = TRY_CAST(t.DT_OPEN AS date)
          AND fk_open.TERM   = t.termdays
WHERE   t.dt_rep       = @Anchor
  AND   t.section_name = N'Срочные'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.out_rub      IS NOT NULL;

/*=======================================================================
  2. spread_final для фиксированных депозитов
=======================================================================*/
IF OBJECT_ID('tempdb..#fix_spread') IS NOT NULL DROP TABLE #fix_spread;

/* нормализуем срок (31±10 → 91) и ищем окно ±15 дн */
WITH m AS (
    SELECT  b.con_id,
            b.out_rub,
            b.termdays,
            b.conv,
            seg = CASE WHEN b.TSEGMENTNAME = N'Розничный Бизнес' THEN 'R' ELSE 'O' END,
            term_use = CASE WHEN b.termdays BETWEEN 21 AND 41 THEN 91
                            ELSE b.termdays END
    FROM    #base b
    WHERE   b.is_floatrate = 0
),
matched AS (
    SELECT  m.con_id,
            r.tobe_end,
            r.key_avg,
            r.term_nom,
            m.conv
    FROM    m
    JOIN    #ref_spread r
           ON r.seg = m.seg
          AND m.term_use BETWEEN r.term_lo AND r.term_hi
)
SELECT  con_id,
        /* если conv = AT_THE_END – вычитаем KEY напрямую;
           иначе переводим TO-BE-ставку в ежемесячную */
        spread_final = CASE
             WHEN conv = 'AT_THE_END'
                  THEN tobe_end - key_avg
             ELSE CAST([LIQUIDITY].[liq].[fnc_IntRate]
                        (tobe_end,'at the end','monthly',term_nom,1) AS decimal(9,6))
                  -  key_avg
        END
INTO    #fix_spread
FROM    matched;

/*=======================================================================
  3. ROLL-OVER-цепочка (spread_fix ← final|fact)
=======================================================================*/
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;

;WITH base2 AS (      /* «обновляем» spread_fix */
        SELECT  b.con_id,
                b.out_rub,
                b.is_floatrate,
                b.termdays,
                b.dt_open,
                spread_float,
                spread_fix = COALESCE(fs.spread_final, b.spread_fix_fact)
        FROM    #base b
        LEFT    JOIN #fix_spread fs ON fs.con_id = b.con_id
),
seq AS (
        SELECT  *, n = 0
        FROM    base2
        UNION ALL
        SELECT  s.con_id, s.out_rub, s.is_floatrate, s.termdays,
                DATEADD(day,s.termdays,s.dt_open),
                s.spread_float,
                s.spread_fix,
                n + 1
        FROM    seq s
        WHERE   DATEADD(day,s.termdays,s.dt_open) <= @HorizonTo
)
SELECT  con_id, out_rub, is_floatrate, termdays,
        dt_open,
        dt_close = DATEADD(day,termdays,dt_open),
        spread_float,
        spread_fix
INTO    #rolls
FROM    seq
OPTION (MAXRECURSION 0);

/*=======================================================================
  4. day-by-day поток ставок
=======================================================================*/
IF OBJECT_ID('tempdb..#daily') IS NOT NULL DROP TABLE #daily;
SELECT  c.d            AS dt_rep,
        w.con_id,
        w.out_rub,
        rate_con = CASE
                      WHEN w.is_floatrate = 1
                           THEN ks.KEY_RATE + w.spread_float
                      ELSE ISNULL(fko.AVG_KEY_RATE + w.spread_fix,
                                  w.spread_fix)
                   END
INTO    #daily
FROM    #cal c
JOIN    #rolls w
          ON c.d BETWEEN w.dt_open AND DATEADD(day,-1,w.dt_close)
LEFT JOIN #key_spot ks
          ON ks.DT_REP = c.d
LEFT JOIN WORK.ForecastKey_Cache fko
          ON fko.DT_REP = w.dt_open
         AND fko.TERM   = w.termdays;

/*=======================================================================
  5. выгрузка в WORK.Forecast_Balance* _v3
=======================================================================*/
IF OBJECT_ID('WORK.Forecast_BalanceDaily_v3','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDaily_v3
    (dt_rep DATE PRIMARY KEY,
     out_rub_total DECIMAL(20,2),
     rate_con      DECIMAL(9,4));
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDaily_v3;

IF OBJECT_ID('WORK.Forecast_BalanceDeals_v3','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDeals_v3
    (dt_rep DATE,
     con_id  BIGINT,
     out_rub DECIMAL(20,2),
     rate_con DECIMAL(9,4),
     CONSTRAINT PK_FBD_v3 PRIMARY KEY(dt_rep,con_id));
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDeals_v3;

/* агрегат */
INSERT INTO WORK.Forecast_BalanceDaily_v3(dt_rep,out_rub_total,rate_con)
SELECT  dt_rep,
        SUM(out_rub),
        SUM(out_rub*rate_con)/SUM(out_rub)
FROM    #daily
GROUP BY dt_rep;

/* деталка (только n=0 и roll-over, без внутренних дней) */
INSERT INTO WORK.Forecast_BalanceDeals_v3(dt_rep,con_id,out_rub,rate_con)
SELECT  w.dt_open AS dt_rep,
        w.con_id,
        w.out_rub,
        CASE WHEN w.is_floatrate = 1
                 THEN ks.KEY_RATE + w.spread_float
             ELSE ISNULL(fko.AVG_KEY_RATE + w.spread_fix, w.spread_fix)
        END
FROM    #rolls w
JOIN    #key_spot ks  ON ks.DT_REP = w.dt_open   -- spot KEY на дату roll-over
LEFT    JOIN WORK.ForecastKey_Cache fko
           ON fko.DT_REP = w.dt_open AND fko.TERM = w.termdays
WHERE   w.dt_open BETWEEN @Anchor AND @HorizonTo;
GO
