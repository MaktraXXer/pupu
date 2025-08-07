ниже — два самостоятельных .sql-блока.
Запускайте их по-очереди (в одной сессии, но скрипт №1 → затем скрипт №2).

⸻

► СКРИПТ № 1 :  FLOAT + FIX-base

(никакого «переворота», только то, что уже корректно работает)

/* ══════════════════════════════════════════════════════════════
   NS-forecast  — Часть 1
   FLOAT  (prod_id 3103)   +   FIX-base  (prod_id 654, ≤ июн-25)
   сохраняет:
       WORK.Forecast_NS_Float
       WORK.Forecast_NS_FixBase
   ═════════════════════════════════════════════════════════════ */
USE ALM_TEST;
GO
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31';

/* — календарь горизонта — */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal)<@HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* — ключевая ставка spot (TERM=1) — */
DROP TABLE IF EXISTS #key;
SELECT DT_REP,KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* — фактический портфель @Anchor — */
DROP TABLE IF EXISTS #bal;
SELECT  t.con_id,t.cli_id,t.prod_id,
        t.out_rub,t.rate_con,
        CAST(t.dt_open  AS date) dt_open,
        CAST(t.dt_close AS date) dt_close,
        t.TSEGMENTNAME
INTO    #bal
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE   t.dt_rep=@Anchor
  AND   t.section_name=N'Накопительный счёт'
  AND   t.block_name  =N'Привлечение ФЛ'
  AND   t.od_flag=1 AND t.cur='810'
  AND   t.out_rub IS NOT NULL
  AND  (t.dt_close_fact IS NULL OR t.dt_close_fact>=@Anchor);

/* ─────────── FLOAT (спред фиксируем в dt_open) ─────────────── */
DROP TABLE IF EXISTS #FLOAT_daily;
SELECT  b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
        c.d dt_rep,
        (b.rate_con-k0.KEY_RATE)+k1.KEY_RATE AS rate_con
INTO    #FLOAT_daily
FROM    #bal b
JOIN    #key k0 ON k0.DT_REP=b.dt_open
JOIN    #cal c  ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
JOIN    #key k1 ON k1.DT_REP=c.d
WHERE   b.prod_id=3103;

DROP TABLE IF EXISTS WORK.Forecast_NS_Float;
SELECT  dt_rep,
        SUM(out_rub)                       out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) rate_avg
INTO    WORK.Forecast_NS_Float
FROM   #FLOAT_daily
GROUP  BY dt_rep;

/* ─────────── FIX-base (ставка константа, ≤ июн-25) ─────────── */
DROP TABLE IF EXISTS #FIX_base_daily;
SELECT  b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
        c.d dt_rep,
        b.rate_con AS rate_con
INTO    #FIX_base_daily
FROM    #bal b
JOIN    #cal c ON c.d BETWEEN b.dt_open AND @HorizonTo
WHERE   b.prod_id=654
  AND   b.dt_open<'2025-07-01';

DROP TABLE IF EXISTS WORK.Forecast_NS_FixBase;
SELECT  dt_rep,
        SUM(out_rub)                       out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) rate_avg
INTO    WORK.Forecast_NS_FixBase
FROM   #FIX_base_daily
GROUP  BY dt_rep;

PRINT N'-- Часть 1 готова: FLOAT + FIX-base';
GO


⸻

► СКРИПТ № 2 :  FIX-promo (2 мес roll + перелив) + Сборка портфеля

/* ══════════════════════════════════════════════════════════════
   NS-forecast  — Часть 2
   FIX-promo (2 full months roll, base-day, client-перелив)
   Требует, чтобы Часть 1 уже создала:
       WORK.Forecast_NS_Float
       WORK.Forecast_NS_FixBase
   Результаты:
       • WORK.Forecast_NS_Promo
       • WORK.Forecast_BalanceDaily_NS  (итог)
   ═════════════════════════════════════════════════════════════ */
USE ALM_TEST;
GO
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/* — календарь + ключ — */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal)<@HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

DROP TABLE IF EXISTS #key;
SELECT DT_REP,KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

DECLARE @KeyAnchor decimal(9,4) =
        (SELECT KEY_RATE FROM #key WHERE DT_REP=@Anchor);

/* — факт-портфель (только promo-NS: июль + август-25) — */
DROP TABLE IF EXISTS #promo_start;
SELECT  t.con_id,t.cli_id,t.prod_id,
        t.out_rub,
        CAST(t.dt_open AS date)  dt_open,
        t.TSEGMENTNAME
INTO    #promo_start
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE   t.dt_rep=@Anchor
  AND   t.prod_id=654
  AND   t.dt_open BETWEEN '2025-07-01' AND '2025-08-31'
  AND   t.cur='810' AND t.od_flag=1 AND t.out_rub IS NOT NULL;

/* ───── 1. СПРЕДЫ ДЛЯ PROMO (фиксируем на @Anchor) ─────────── */
DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

WITH aug AS (
     SELECT TSEGMENTNAME,
            SUM(out_rub*rate_con)/SUM(out_rub) AS w_rate
     FROM   ALM.ALM.vw_balance_rest_all WITH(NOLOCK)
     WHERE  dt_rep=@Anchor
       AND  prod_id=654
       AND  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
     GROUP  BY TSEGMENTNAME)
SELECT @Spread_DChbo  = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО'            THEN w_rate END)-@KeyAnchor,
       @Spread_Retail = MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END)-@KeyAnchor
FROM   aug;

PRINT N'=== spread (модель) ===';
SELECT N'ДЧБО' AS seg,@Spread_DChbo AS spread UNION ALL
SELECT N'Розничный бизнес',@Spread_Retail;
PRINT ' ';

/* ───── 2. PROMO-ставка каждого месяца (мин KEY + spread) ──── */
DROP TABLE IF EXISTS #promo_rate_month;
WITH kmin AS (
     SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) m1,
            MIN(KEY_RATE) kmin
     FROM   #key
     GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
SELECT  m1,
        N'ДЧБО'              AS seg,
        kmin+@Spread_DChbo   AS promo_rate
INTO    #promo_rate_month
FROM   kmin
UNION ALL
SELECT  m1,N'Розничный бизнес',kmin+@Spread_Retail FROM kmin;

PRINT N'=== promo-rate by month ===';
SELECT * FROM #promo_rate_month ORDER BY m1,seg;
PRINT ' ';

/* ───── 3. рекурсивная сетка promo-окон (2 полных месяца) ──── */
DROP TABLE IF EXISTS #promo_win;
;WITH seed AS (
      SELECT  p.con_id,p.cli_id,p.TSEGMENTNAME,p.out_rub,
              win_start = p.dt_open,
              win_end   = DATEADD(day,-1,DATEADD(month,2,p.dt_open))  -- 2 мес –1 д
      FROM    #promo_start p )
, seq AS (
      SELECT *,0 n FROM seed
      UNION ALL
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,
             DATEADD(day,1,win_end),
             DATEADD(day,-1,DATEADD(month,2,DATEADD(day,1,win_end))),
             n+1
      FROM   seq
      WHERE  DATEADD(day,1,win_end)<=@HorizonTo)
SELECT * INTO #promo_win FROM seq OPTION (MAXRECURSION 0);

/* ───── 4. дневная лента (ставка фикс. на весь promo-период) ─ */
DROP TABLE IF EXISTS #promo_daily_raw;
--   рассчитываем promo-rate УЖЕ при генерации окна,
--   а не день-за-днём, чтобы она не «ездило» вместе с KEY
SELECT  w.con_id,w.cli_id,w.TSEGMENTNAME,w.out_rub,
        c.d dt_rep,
        CASE
            WHEN c.d <  w.win_end+1 THEN     -- promo-дни
                 CASE w.TSEGMENTNAME
                       WHEN N'ДЧБО'            THEN @Spread_DChbo  + @KeyAnchor
                       ELSE                         @Spread_Retail + @KeyAnchor
                 END
            WHEN c.d = w.win_end+1 THEN @BaseRate   -- базовый день
        END AS rate_con
INTO    #promo_daily_raw
FROM    #promo_win w
JOIN    #cal c
       ON c.d BETWEEN w.win_start AND w.win_end+1;   -- +1 = base-day

/* ───── 5. перелив         (только promo-счета, только 1-е число) ─ */
DROP TABLE IF EXISTS #promo_1st;
SELECT  cli_id,
        m1=DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
INTO    #promo_1st
FROM   #promo_daily_raw
WHERE  dt_rep=DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
GROUP  BY cli_id,DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1);

DROP TABLE IF EXISTS #promo_1st_glue;
SELECT  NULL       AS con_id,
        p.cli_id,
        NULL       AS TSEGMENTNAME,
        SUM(d.out_rub)                     AS out_rub,
        p.m1       AS dt_rep,
        MAX(d.rate_con)                    AS rate_con
INTO    #promo_1st_glue
FROM    #promo_1st p
JOIN    #promo_daily_raw d
       ON d.cli_id=p.cli_id AND d.dt_rep=p.m1
GROUP  BY p.cli_id,p.m1;

/* без 1-го числа (чтобы избежать дубля объёмов) */
DROP TABLE IF EXISTS #promo_not1;
SELECT * INTO #promo_not1
FROM   #promo_daily_raw
WHERE  dt_rep<>DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1);

/* ───── 6. агрегат PROMO для портфеля ───────────────────────── */
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;
SELECT  dt_rep,
        SUM(out_rub)                       out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) rate_avg
INTO    WORK.Forecast_NS_Promo
FROM (
      SELECT * FROM #promo_not1
      UNION ALL
      SELECT * FROM #promo_1st_glue) z
GROUP  BY dt_rep;

PRINT N'-- Часть 2: PROMO ready';
PRINT ' ';

/* ══════════════════════════════════════════════════════════════
   7. Финальная склейка  (FLOAT + FIX-base + PROMO)
═════════════════════════════════════════════════════════════*/
DROP TABLE IF EXISTS WORK.Forecast_BalanceDaily_NS;
SELECT  f.dt_rep,
        f.out_rub_total+b.out_rub_total+p.out_rub_total                           AS out_rub_total,
        (f.out_rub_total*f.rate_avg +
         b.out_rub_total*b.rate_avg +
         p.out_rub_total*p.rate_avg)
/        NULLIF(f.out_rub_total+b.out_rub_total+p.out_rub_total,0)                AS rate_avg
INTO    WORK.Forecast_BalanceDaily_NS
FROM   WORK.Forecast_NS_Float   f
JOIN   WORK.Forecast_NS_FixBase b ON b.dt_rep=f.dt_rep
JOIN   WORK.Forecast_NS_Promo   p ON p.dt_rep=f.dt_rep;

/* ───── 8. Краткий sanity-чек ───────────────────────────────── */
PRINT N'╔ sample «ступеньки» ════════════════════════════════════';
SELECT dt_rep,out_rub_total,rate_avg
FROM   WORK.Forecast_BalanceDaily_NS
WHERE  dt_rep IN ('2025-08-30','2025-08-31',
                  '2025-09-01','2025-09-15',
                  '2025-09-30','2025-10-01')
ORDER  BY dt_rep;
PRINT N'╚════════════════════════════════════════════════════════';

PRINT N'=== FLOAT TOP-5 ===';   SELECT TOP 5 * FROM WORK.Forecast_NS_Float   ORDER BY dt_rep;
PRINT N'=== FIX-base TOP-5 ===';SELECT TOP 5 * FROM WORK.Forecast_NS_FixBase ORDER BY dt_rep;
PRINT N'=== PROMO TOP-5 ===';   SELECT TOP 5 * FROM WORK.Forecast_NS_Promo   ORDER BY dt_rep;
PRINT N'=== portfolio TOP-20 ===';
SELECT TOP 20 * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;
GO


⸻

что изменилось в PROMO-логике (блок 2):

было m4	стало (этот скрипт)
ставка promo пересчитывалась день-в-день через текущий KEY	ставка фиксируется на весь promo-период (KEY @Anchor + spread)
base-day ошибочно попадал на 1-е число	base-day = последний календарный день promo-окна
дубли объёмов из-за glue	оригинальные строки на 1-е число удаляются, остаётся одна склейка
«скачок» объёма 1-го числа	объём promo не меняется – только перераспределяется внутри клиента

проверяйте выборки PROMO TOP-5 и «ступеньки»:
должно быть 127 млрд с 6.5 % / 17 - 17.3 % именно там, где вы ожидали.
