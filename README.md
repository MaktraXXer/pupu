ок, делаем «двухэтажку»:
Part 1 = RATES (память ставок по счёту на ключевые даты) → сохраняем в WORK.NS_RatesKeyDates
Part 2 = ALLOC (переливы объёмов по ключевым датам + развёртка в дни) → итог WORK.Forecast_NS_Promo

Ниже — два полностью автономных скрипта. Сначала запускаешь Part 1 (строит и сохраняет ставки), затем Part 2 (читает ставки, выбирает победителей, переливает объёмы и строит суточную ленту со step-логикой: между ключевыми датами берётся последнее актуальное значение).

⸻

Part 1 — RATES: «память ставок» по счёту на ключевые даты

/* ============================================================
   PART 1: NS (prod_id = 654) — RATES ONLY (key-dates memory)
   Сохраняет ставки по каждому кону на ключевые даты в WORK.NS_RatesKeyDates
   Ключевые даты:
      - @Anchor
      - все 1-е числа между @Anchor..@HorizonTo
      - все последние дни месяца (EOM) между @Anchor..@HorizonTo
   ============================================================ */

USE ALM_TEST;
SET NOCOUNT ON;

/* ---------- параметры ---------- */
DECLARE
    @Scenario   tinyint      = 1,
    @Anchor     date         = '2025-11-03',
    @HorizonTo  date         = '2026-03-31',
    @BaseRate   decimal(9,4) = 0.0450,  -- базовая ставка (на base_day/EOM)
    @UseFixedPromoFromNextMonth bit = 0, -- «фикс-промо» только в месяце, следующем за якорем
    @UseHardPromoSpread bit = 0;         -- жёсткие спреды вместо расчётных

/* окно фикс-промо: ровно месяц после якоря */
DECLARE @FixedPromoStart date = DATEADD(day,1,EOMONTH(@Anchor));   -- 1-е след. месяца
DECLARE @FixedPromoEnd   date = DATEADD(month,1,@FixedPromoStart); -- 1-е через месяц
/* старт жёстких спредов */
DECLARE @HardSpreadStart date = @Anchor;

/* ---------- календарь ключевых дат ---------- */
IF OBJECT_ID('tempdb..#cal_all') IS NOT NULL DROP TABLE #cal_all;
;WITH d AS (
  SELECT d = @Anchor
  UNION ALL SELECT DATEADD(day,1,d) FROM d WHERE DATEADD(day,1,d) <= @HorizonTo
)
SELECT d INTO #cal_all FROM d OPTION (MAXRECURSION 0);

IF OBJECT_ID('tempdb..#cal_key') IS NOT NULL DROP TABLE #cal_key;
;WITH key1 AS ( SELECT d FROM #cal_all WHERE DAY(d)=1 OR d=@Anchor ),
     key2 AS ( SELECT d=EOMONTH(d) FROM #cal_all )
SELECT DISTINCT d
INTO #cal_key
FROM (
  SELECT d FROM key1
  UNION
  SELECT d FROM key2
) u;

/* ---------- ключевая ставка (TERM=1) ---------- */
IF OBJECT_ID('tempdb..#key') IS NOT NULL DROP TABLE #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@Scenario AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* ---------- промо ставки / спреды на якоре ---------- */
DECLARE
    @PromoRate_DChbo  decimal(9,4) = 0.1600,
    @PromoRate_Retail decimal(9,4) = 0.1590;

DECLARE @KeyAtAnchor decimal(9,4);
SELECT @KeyAtAnchor = KEY_RATE FROM #key WHERE DT_REP=@Anchor;

DECLARE
    @Spread_DChbo  decimal(9,4) = @PromoRate_DChbo  - @KeyAtAnchor,
    @Spread_Retail decimal(9,4) = @PromoRate_Retail - @KeyAtAnchor;

DECLARE
    @HardSpread_DChbo  decimal(9,4) = -0.0030,
    @HardSpread_Retail decimal(9,4) = -0.0040;

/* ---------- снапшот промо-портфеля на якоре (#bal_prom) ---------- */
DECLARE
    @OpenFrom date = DATEFROMPARTS(YEAR(DATEADD(month,-1,@Anchor)), MONTH(DATEADD(month,-1,@Anchor)), 1),
    @OpenTo   date = EOMONTH(@Anchor);

IF OBJECT_ID('tempdb..#bal_src') IS NOT NULL DROP TABLE #bal_src;
SELECT
    t.dt_rep,
    CAST(t.dt_open  AS date)          AS dt_open,
    CAST(t.dt_close AS date)          AS dt_close,
    CAST(t.con_id   AS bigint)        AS con_id,
    CAST(t.cli_id   AS bigint)        AS cli_id,
    CAST(t.prod_id  AS int)           AS prod_id,
    CAST(t.out_rub  AS decimal(20,2)) AS out_rub,
    CAST(t.rate_con AS decimal(9,4))  AS rate_balance,
    t.rate_con_src,
    t.TSEGMENTNAME,
    CAST(r.rate     AS decimal(9,4))  AS rate_liq
INTO #bal_src
FROM   alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = t.con_id
       AND CASE WHEN t.dt_open=t.dt_rep
                THEN DATEADD(day,1,t.dt_rep)
                ELSE t.dt_rep
           END BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day,2,@Anchor)
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.cur          = '810'
  AND  t.prod_id      = 654;

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src(con_id, dt_rep);

IF OBJECT_ID('tempdb..#bal_prom') IS NOT NULL DROP TABLE #bal_prom;
;WITH bal_pos AS (
    SELECT *,
           MIN(CASE WHEN rate_balance>0 AND rate_con_src=N'счет ультра,вручную'
                    THEN rate_balance END)
           OVER (PARTITION BY con_id
                 ORDER BY dt_rep
                 ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
    FROM #bal_src
    WHERE dt_rep=@Anchor
),
rate_calc AS (
    SELECT *,
      CASE
        WHEN rate_liq IS NULL THEN
             CASE WHEN rate_balance<0 THEN COALESCE(rate_pos, rate_balance)
                  ELSE rate_balance END
        WHEN rate_liq <0 AND rate_balance>0 THEN rate_balance
        WHEN rate_liq <0 AND rate_balance<0 THEN COALESCE(rate_pos, rate_balance)
        WHEN rate_liq>=0 AND rate_balance>=0 THEN rate_liq
        WHEN rate_liq> 0 AND rate_balance<0 THEN rate_liq
        ELSE rate_liq
      END AS rate_use
    FROM bal_pos
)
SELECT
  con_id, cli_id, out_rub,
  rate_anchor = CAST(rate_use AS decimal(9,4)),
  dt_open,
  TSEGMENTNAME
INTO #bal_prom
FROM rate_calc
WHERE out_rub IS NOT NULL
  AND dt_open BETWEEN @OpenFrom AND @OpenTo;

IF OBJECT_ID('tempdb..#z0') IS NOT NULL DROP TABLE #z0;
SELECT
  CAST(z.con_id AS bigint) AS con_id,
  CAST(z.cli_id AS bigint) AS cli_id,
  CAST(0.00     AS decimal(20,2)) AS out_rub,
  CAST(COALESCE(z.rate_con,
       CASE WHEN LTRIM(RTRIM(COALESCE(z.TSEGMENTNAME,N'')))=N'ДЧБО'
            THEN @PromoRate_DChbo ELSE @PromoRate_Retail END) AS decimal(9,4)) AS rate_anchor,
  CAST(z.dt_open AS date) AS dt_open,
  CASE WHEN LTRIM(RTRIM(COALESCE(z.TSEGMENTNAME,N'')))=N''
       THEN N'Розничный бизнес' ELSE z.TSEGMENTNAME END AS TSEGMENTNAME
INTO #z0
FROM ALM.ehd.import_ZeroContracts z WITH (NOLOCK)
WHERE z.dt_open<=@Anchor
  AND NOT EXISTS(SELECT 1 FROM #bal_prom p WHERE p.con_id=z.con_id);

INSERT #bal_prom(con_id,cli_id,out_rub,rate_anchor,dt_open,TSEGMENTNAME)
SELECT con_id,cli_id,out_rub,rate_anchor,dt_open,TSEGMENTNAME
FROM #z0;

CREATE UNIQUE CLUSTERED INDEX IX_bal_prom ON #bal_prom(con_id);

/* ---------- цикл окна по договору ---------- */
IF OBJECT_ID('tempdb..#cycles') IS NOT NULL DROP TABLE #cycles;
;WITH seed AS (
  SELECT b.con_id, b.cli_id, b.TSEGMENTNAME, b.out_rub,
         CAST(0 AS int) AS cycle_no,
         b.dt_open AS win_start,
         EOMONTH(DATEADD(month,1,b.dt_open))                  AS base_day,
         DATEADD(day,-1,EOMONTH(DATEADD(month,1,b.dt_open)))  AS promo_end
  FROM #bal_prom b
),
seq AS (
  SELECT * FROM seed
  UNION ALL
  SELECT s.con_id, s.cli_id, s.TSEGMENTNAME, s.out_rub,
         s.cycle_no+1,
         DATEADD(day,1,s.base_day),
         EOMONTH(DATEADD(month,1,DATEADD(day,1,s.base_day))),
         DATEADD(day,-1,EOMONTH(DATEADD(month,1,DATEADD(day,1,s.base_day))))
  FROM seq s
  WHERE DATEADD(day,1,s.base_day) <= @HorizonTo
)
SELECT * INTO #cycles FROM seq OPTION (MAXRECURSION 0);

CREATE UNIQUE CLUSTERED INDEX IX_cycles ON #cycles(con_id, cycle_no);

/* ---------- ставки на КЛЮЧЕВЫЕ даты (только память ставок) ---------- */
IF OBJECT_ID('WORK.NS_RatesKeyDates','U') IS NOT NULL DROP TABLE WORK.NS_RatesKeyDates;

;WITH kspan AS (
  /* связка: для каждого con_id берём ключевые даты, которые попадают в его окне */
  SELECT c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
         k.d AS dt_key
  FROM   #cycles c
  JOIN   #cal_key k
    ON   k.d BETWEEN c.win_start AND c.base_day
), rk AS (
  SELECT
    x.con_id, x.cli_id, x.TSEGMENTNAME, x.cycle_no, x.win_start, x.promo_end, x.base_day, x.dt_key,
    rate_con =
    CASE
      WHEN x.dt_key = x.base_day THEN @BaseRate
      WHEN DAY(x.dt_key) = 1 THEN
           CASE
             WHEN @UseFixedPromoFromNextMonth=1
              AND x.dt_key>=@FixedPromoStart AND x.dt_key<@FixedPromoEnd THEN
                CASE x.TSEGMENTNAME WHEN N'ДЧБО' THEN @PromoRate_DChbo ELSE @PromoRate_Retail END
             WHEN @UseHardPromoSpread=1 AND x.dt_key>=@HardSpreadStart THEN
                CASE WHEN x.cycle_no=0
                      THEN bp.rate_anchor
                      ELSE CASE x.TSEGMENTNAME
                             WHEN N'ДЧБО' THEN @HardSpread_DChbo  + k.KEY_RATE
                             ELSE               @HardSpread_Retail+ k.KEY_RATE
                           END
                END
             ELSE
                CASE WHEN x.cycle_no=0
                      THEN bp.rate_anchor
                      ELSE CASE x.TSEGMENTNAME
                             WHEN N'ДЧБО' THEN @Spread_DChbo  + k.KEY_RATE
                             ELSE               @Spread_Retail+ k.KEY_RATE
                           END
                END
           END
      ELSE
           /* якорь/прочие ключевые даты внутри окна, не 1-е и не EOM:
              используем тот же приоритет, что и для «обычных промо-дней» */
           CASE
             WHEN @UseFixedPromoFromNextMonth=1
              AND x.dt_key>=@FixedPromoStart AND x.dt_key<@FixedPromoEnd THEN
                CASE x.TSEGMENTNAME WHEN N'ДЧБО' THEN @PromoRate_DChbo ELSE @PromoRate_Retail END
             WHEN @UseHardPromoSpread=1 AND x.dt_key>=@HardSpreadStart THEN
                CASE WHEN x.cycle_no=0
                      THEN bp.rate_anchor
                      ELSE CASE x.TSEGMENTNAME
                             WHEN N'ДЧБО' THEN @HardSpread_DChbo  + k.KEY_RATE
                             ELSE               @HardSpread_Retail+ k.KEY_RATE
                           END
                END
             ELSE
                CASE WHEN x.cycle_no=0
                      THEN bp.rate_anchor
                      ELSE CASE x.TSEGMENTNAME
                             WHEN N'ДЧБО' THEN @Spread_DChbo  + k.KEY_RATE
                             ELSE               @Spread_Retail+ k.KEY_RATE
                           END
                END
           END
    END
  FROM kspan x
  LEFT JOIN #key k       ON k.DT_REP = x.dt_key
  JOIN      #bal_prom bp ON bp.con_id = x.con_id
)
SELECT
  con_id, cli_id, TSEGMENTNAME, cycle_no, win_start, promo_end, base_day,
  dt_key, CAST(rate_con AS decimal(9,4)) AS rate_con
INTO WORK.NS_RatesKeyDates
FROM rk;

CREATE UNIQUE CLUSTERED INDEX IX_NS_RatesKeyDates ON WORK.NS_RatesKeyDates(con_id, dt_key);
CREATE INDEX IX_NS_RatesKeyDates_cli ON WORK.NS_RatesKeyDates(cli_id, dt_key);


⸻

Part 2 — ALLOC: переливы объёмов по ключевым датам + развёртка в дни (step)

/* ============================================================
   PART 2: NS (prod_id = 654) — ALLOC (volume rebalancing)
   Читает WORK.NS_RatesKeyDates и строит дневную ленту:
      - перераспределение Σ клиента на base_day (EOM)
      - перераспределение Σ клиента на каждое 1-е
      - между ключевыми датами держим «последнего победителя»
      - до первого события после Anchor держим историческое распределение
   Результат: WORK.Forecast_NS_Promo (dt_rep, out_rub_total, rate_avg)
   ============================================================ */

USE ALM_TEST;
SET NOCOUNT ON;

/* ---------- параметры (должны совпадать с Part 1) ---------- */
DECLARE
    @Anchor     date         = '2025-11-03',
    @HorizonTo  date         = '2026-03-31';

/* ---------- календарь (все дни) ---------- */
IF OBJECT_ID('tempdb..#cal_all','U') IS NOT NULL DROP TABLE #cal_all;
;WITH d AS (
  SELECT d=@Anchor
  UNION ALL SELECT DATEADD(day,1,d) FROM d WHERE DATEADD(day,1,d) <= @HorizonTo
)
SELECT d INTO #cal_all FROM d OPTION (MAXRECURSION 0);

/* ---------- ключевые даты ---------- */
IF OBJECT_ID('tempdb..#cal_key','U') IS NOT NULL DROP TABLE #cal_key;
;WITH key1 AS ( SELECT d FROM #cal_all WHERE DAY(d)=1 OR d=@Anchor ),
     key2 AS ( SELECT d=EOMONTH(d) FROM #cal_all )
SELECT DISTINCT d
INTO #cal_key
FROM (
  SELECT d FROM key1
  UNION
  SELECT d FROM key2
) u;

/* ---------- снимаем снапшот распределения объёмов на Anchor ---------- */
IF OBJECT_ID('tempdb..#bal_prom','U') IS NOT NULL DROP TABLE #bal_prom;
SELECT con_id, cli_id, out_rub, dt_open, TSEGMENTNAME, rate_anchor
INTO #bal_prom
FROM (
  /* берём то же, что готовили в Part 1 (можно материализовать в отдельную таблицу, если нужно) */
  SELECT bp.con_id, bp.cli_id, bp.out_rub, bp.dt_open, bp.TSEGMENTNAME, bp.rate_anchor
  FROM   tempdb.sys.objects o               -- заглушка, чтобы блок был синтаксически замкнут
  CROSS APPLY (SELECT 1 AS _n) dummy
  JOIN   (SELECT * FROM #bal_prom) bp       -- при отдельном запуске Part 2: замените на сохранённый снапшот!
) s;

/* ЕСЛИ Part 2 запускается отдельно от Part 1:
   — замените блок выше на чтение материализованного снапшота, напр.:
   SELECT con_id, cli_id, out_rub, dt_open, TSEGMENTNAME, rate_anchor
   INTO #bal_prom
   FROM WORK.NS_BalPromoAnchor;   -- если будете сохранять в Part 1
*/
-- Индексы
CREATE UNIQUE CLUSTERED INDEX IX_bal_prom ON #bal_prom(con_id);
CREATE INDEX IX_bal_prom_cli ON #bal_prom(cli_id);

/* ---------- «память ставок» на ключевых датах ---------- */
IF OBJECT_ID('tempdb..#rates_key','U') IS NOT NULL DROP TABLE #rates_key;
SELECT *
INTO #rates_key
FROM WORK.NS_RatesKeyDates
WHERE dt_key BETWEEN @Anchor AND @HorizonTo;

CREATE UNIQUE CLUSTERED INDEX IX_rates_key ON #rates_key(con_id, dt_key);
CREATE INDEX IX_rates_key_cli ON #rates_key(cli_id, dt_key);

/* ---------- определяем base_day по кону (из #rates_key) ---------- */
IF OBJECT_ID('tempdb..#base_map','U') IS NOT NULL DROP TABLE #base_map;
SELECT DISTINCT con_id, cli_id, base_day
INTO #base_map
FROM #rates_key;

/* ---------- победители на base_day (Σ клиента) ---------- */
IF OBJECT_ID('tempdb..#alloc_base','U') IS NOT NULL DROP TABLE #alloc_base;
;WITH pool AS (
  SELECT r.cli_id, r.con_id, r.dt_key AS dt_rep, r.rate_con
  FROM   #rates_key r
  JOIN   #base_map  b ON b.con_id=r.con_id AND r.dt_key=b.base_day
),
ranked AS (
  SELECT p.*, ROW_NUMBER() OVER(PARTITION BY p.cli_id, p.dt_rep ORDER BY p.rate_con DESC, p.con_id) AS rn
  FROM pool p
),
cli_sum AS (
  SELECT cli_id, SUM(out_rub) AS out_rub_sum FROM #bal_prom GROUP BY cli_id
)
SELECT r.cli_id, r.con_id, r.dt_rep, out_rub = CASE WHEN r.rn=1 THEN cs.out_rub_sum ELSE CAST(0.00 AS decimal(20,2)) END
INTO #alloc_base
FROM ranked r
JOIN cli_sum cs ON cs.cli_id=r.cli_id;

CREATE CLUSTERED INDEX IX_alloc_base ON #alloc_base(cli_id, dt_rep, con_id);

/* ---------- кандидаты на 1-е числа: победители по ставке на 1-е ---------- */
IF OBJECT_ID('tempdb..#alloc_firstday','U') IS NOT NULL DROP TABLE #alloc_firstday;
;WITH m1 AS (
  SELECT DISTINCT
         r.cli_id, r.con_id,
         r.dt_key AS dt_rep
  FROM   #rates_key r
  WHERE  DAY(r.dt_key)=1
),
ranked AS (
  SELECT m1.*, r.rate_con,
         ROW_NUMBER() OVER(PARTITION BY m1.cli_id, m1.dt_rep ORDER BY r.rate_con DESC, m1.con_id) AS rn
  FROM   m1
  JOIN   #rates_key r ON r.con_id=m1.con_id AND r.dt_key=m1.dt_rep
),
cli_sum AS (SELECT cli_id, SUM(out_rub) AS out_rub_sum FROM #bal_prom GROUP BY cli_id)
SELECT r.cli_id, r.con_id, r.dt_rep,
       out_rub = CASE WHEN r.rn=1 THEN cs.out_rub_sum ELSE CAST(0.00 AS decimal(20,2)) END
INTO #alloc_firstday
FROM ranked r
JOIN cli_sum cs ON cs.cli_id=r.cli_id;

CREATE CLUSTERED INDEX IX_alloc_firstday ON #alloc_firstday(cli_id, dt_rep, con_id);

/* ---------- все события перелива (key events) ---------- */
IF OBJECT_ID('tempdb..#alloc_events','U') IS NOT NULL DROP TABLE #alloc_events;
SELECT cli_id, con_id, dt_rep, out_rub
INTO   #alloc_events
FROM (
  SELECT * FROM #alloc_base
  UNION ALL
  SELECT * FROM #alloc_firstday
) u;

CREATE CLUSTERED INDEX IX_alloc_events ON #alloc_events(cli_id, dt_rep, con_id);

/* ---------- первая дата события по клиенту ---------- */
IF OBJECT_ID('tempdb..#first_evt','U') IS NOT NULL DROP TABLE #first_evt;
SELECT cli_id, first_evt = MIN(dt_rep)
INTO   #first_evt
FROM   #alloc_events
GROUP  BY cli_id;

CREATE UNIQUE CLUSTERED INDEX IX_first_evt ON #first_evt(cli_id);

/* ---------- развёртка ALLOC в дни:
   - до first_evt: держим ИСТОРИЧЕСКОЕ распределение (#bal_prom) по всем con_id
   - после/включая first_evt: держим «последнего победителя» (Σ клиента) ---------- */
IF OBJECT_ID('tempdb..#alloc_daily','U') IS NOT NULL DROP TABLE #alloc_daily;

;WITH day_cli AS (
  SELECT c.d AS dt_rep, b.cli_id
  FROM   #cal_all c
  JOIN  (SELECT DISTINCT cli_id FROM #bal_prom) b ON 1=1
),
last_evt AS (
  /* последняя запись события на дату */
  SELECT dc.dt_rep, dc.cli_id, e.con_id, e.out_rub,
         ROW_NUMBER() OVER(PARTITION BY dc.cli_id, dc.dt_rep ORDER BY e.dt_rep DESC, e.con_id) AS rn
  FROM day_cli dc
  OUTER APPLY (
      SELECT TOP (1) * FROM #alloc_events a
      WHERE a.cli_id=dc.cli_id AND a.dt_rep <= dc.dt_rep
      ORDER BY a.dt_rep DESC, a.con_id
  ) e
),
pre_init AS (
  /* признак: дата < первого события → используем исходное распределение */
  SELECT dc.dt_rep, dc.cli_id,
         CASE WHEN dc.dt_rep < fe.first_evt THEN 1 ELSE 0 END AS use_initial
  FROM day_cli dc
  LEFT JOIN #first_evt fe ON fe.cli_id=dc.cli_id
),
initial_rows AS (
  /* до первого события: исходные объёмы по КАЖДОМУ счёту клиента */
  SELECT p.cli_id, p.con_id, d.dt_rep, p.out_rub
  FROM   pre_init d
  JOIN   #bal_prom p ON p.cli_id=d.cli_id
  WHERE  d.use_initial=1
),
winner_rows AS (
  /* с момента первого события: один победитель держит Σ */
  SELECT le.dt_rep, le.cli_id, le.con_id, le.out_rub
  FROM   last_evt le
  WHERE  le.rn=1  -- берём единственного «последнего»
    AND  le.out_rub IS NOT NULL
),
alloc_union AS (
  SELECT * FROM initial_rows
  UNION ALL
  SELECT cli_id, con_id, dt_rep, out_rub FROM winner_rows
)
SELECT dt_rep, cli_id, con_id, out_rub
INTO   #alloc_daily
FROM   alloc_union;

CREATE CLUSTERED INDEX IX_alloc_daily ON #alloc_daily(dt_rep, cli_id, con_id);

/* ---------- ставки в дни: берём ПОСЛЕДНЮЮ ключевую дату <= dt_rep (память ставок) ---------- */
IF OBJECT_ID('tempdb..#rates_daily','U') IS NOT NULL DROP TABLE #rates_daily;
;WITH rspan AS (
  SELECT ad.dt_rep, ad.con_id
  FROM   #alloc_daily ad
  GROUP BY ad.dt_rep, ad.con_id
),
rr AS (
  SELECT rs.dt_rep, rs.con_id, rk.rate_con
  FROM   rspan rs
  OUTER APPLY (
     SELECT TOP (1) rate_con
     FROM   #rates_key rk
     WHERE  rk.con_id=rs.con_id AND rk.dt_key <= rs.dt_rep
     ORDER BY rk.dt_key DESC
  ) rk
)
SELECT dt_rep, con_id, CAST(rate_con AS decimal(9,4)) AS rate_con
INTO   #rates_daily
FROM   rr;

CREATE UNIQUE CLUSTERED INDEX IX_rates_daily ON #rates_daily(dt_rep, con_id);

/* ---------- итог: агрегируем по дню ---------- */
IF OBJECT_ID('WORK.Forecast_NS_Promo','U') IS NOT NULL DROP TABLE WORK.Forecast_NS_Promo;

SELECT
  a.dt_rep,
  out_rub_total = SUM(a.out_rub),
  rate_avg      = SUM(a.out_rub * r.rate_con) / NULLIF(SUM(a.out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM #alloc_daily a
JOIN #rates_daily r ON r.con_id=a.con_id AND r.dt_rep=a.dt_rep
GROUP BY a.dt_rep;

/* При необходимости — быстрый контроль (можно убрать): */
-- SELECT TOP (200) * FROM WORK.Forecast_NS_Promo ORDER BY dt_rep;


⸻

Что именно делает эта архитектура
	•	Part 1 (RATES) сохраняет только ставки на ключевых датах по каждому счёту (con_id).
Ставка «помнится» по счёту, не зависит от перелива объёма.
Таблица: WORK.NS_RatesKeyDates(con_id, cli_id, dt_key, rate_con, win_start, promo_end, base_day, …).
	•	Part 2 (ALLOC):
	•	На EOM (base_day) выбирает победителя по ставке этого счёта из NS_RatesKeyDates и переливает Σ клиента на него.
	•	На каждое 1-е число — аналогично (победитель по ставке счёта на 1-е).
	•	Между событиями — держим последнего победителя (step-логика).
	•	До первого события после Anchor — держим историческое распределение по счётам из снапшота.
	•	Для расчёта дневной ставки используем «память ставок»: последняя ключевая дата ≤ день для данного con_id.

Почему это снимает все конфликтные кейсы
	•	«Ставка победителя размазывается на все счета» — невозможно: ставки счётов живут в NS_RatesKeyDates и подтягиваются по con_id.
	•	«Забыли ставки по счёту, который сегодня проиграл» — невозможно: они сохранены независимо от аллокаций.
	•	«Дневные данные слишком тяжёлые» — хранится компактная «память» на ключевых датах; суточная лента строится через step-развёртку (APPLY к последней ключевой дате).

Хочешь — сделаю доп. сохранение снапшота #bal_prom в WORK.NS_BalPromoAnchor, чтобы Part 2 запускался вообще в отрыве от Part 1 без правок.
