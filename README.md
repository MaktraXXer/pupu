ок, делаем нормально: один монолит, без sys.*, без двусмысленностей, без дубликатов.
Логика:
	•	Сначала строим интервалы ставок по каждому счёту только на ключевых датах (якорь, все EOM и все 1-е), затем превращаем их в (dt_from, dt_to).
	•	Потом на событиях (только EOM и 1-е) выбираем победителя по ставке (тай-брейк con_id), строим непересекающиеся сегменты [start; end] помесячно.
	•	Из сегментов формируем дневные аллокации:
до первого события — держим якорный сплит по con_id,
после — весь Σ клиента на победителе сегмента.
Ставка на день — из интервалов (мы никогда не «забываем» ставку конкретного счёта).

В конце получаем:
	•	WORK.NS_BalPromoAnchor — сплит на якоре;
	•	WORK.NS_RateIntervals — интервалы ставок по каждому счёту;
	•	WORK.NS_AllocEvents — только EOM/1-е с победителем;
	•	WORK.Forecast_NS_Promo — дневная агрегированная лента (Σ объёма, средневзвешенная ставка).

Вот полный скрипт от начала до конца:

/* ===============================================================
   NS-forecast — Monolith (FIX-promo-roll, prod_id = 654)
   Без sys.*, без рекурсий по дням, без дубликатов.
   Авторская логика: интервалы ставок → события → сегменты → дни.
   =============================================================== */
USE ALM_TEST;
SET NOCOUNT ON;

/* ---------- ПАРАМЕТРЫ ---------- */
DECLARE
    @Scenario   tinyint      = 1,
    @Anchor     date         = '2025-10-29',
    @HorizonTo  date         = '2026-03-31',
    @BaseRate   decimal(9,4) = 0.0450,      -- ставка на base_day (EOM)
    @UseFixedPromoFromNextMonth bit = 0,    -- фикс-промо в мес. после якоря
    @UseHardPromoSpread        bit = 0;     -- жёсткий спред вместо расчётного

DECLARE @FixedPromoStart date = DATEADD(day,1,EOMONTH(@Anchor));
DECLARE @FixedPromoEnd   date = DATEADD(month,1,@FixedPromoStart);
DECLARE @HardSpreadStart date = @Anchor;

/* ---------- Ключевая ставка (TERM=1) ---------- */
IF OBJECT_ID('tempdb..#key') IS NOT NULL DROP TABLE #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@Scenario AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

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

/* ---------- Снапшот договоров к якорю + нулевые ---------- */
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
FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
LEFT JOIN LIQUIDITY.liq.DepositContract_Rate r
  ON r.con_id = t.con_id
 AND CASE WHEN t.dt_open=t.dt_rep THEN DATEADD(day,1,t.dt_rep) ELSE t.dt_rep END
     BETWEEN r.dt_from AND r.dt_to
WHERE t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day,2,@Anchor)
  AND t.section_name = N'Накопительный счёт'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.prod_id      = 654;

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src(con_id, dt_rep);

IF OBJECT_ID('tempdb..#bal_prom') IS NOT NULL DROP TABLE #bal_prom;
;WITH bal_pos AS (
    SELECT *,
           MIN(CASE WHEN rate_balance>0 AND rate_con_src=N'счет ультра,вручную'
                    THEN rate_balance END)
           OVER (PARTITION BY con_id ORDER BY dt_rep
                 ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
    FROM #bal_src
    WHERE dt_rep=@Anchor
),
rate_calc AS (
    SELECT *,
      CASE
        WHEN rate_liq IS NULL THEN CASE WHEN rate_balance<0 THEN COALESCE(rate_pos, rate_balance)
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

CREATE UNIQUE CLUSTERED INDEX IX_bal_prom ON #bal_prom(con_id);

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
SELECT con_id,cli_id,out_rub,rate_anchor,dt_open,TSEGMENTNAME FROM #z0;

/* Σ на Anchor (для контролек) */
DECLARE @AnchorTotal decimal(20,2) = (SELECT SUM(out_rub) FROM #bal_prom);

/* ---------- Сохраняем сплит якоря ---------- */
IF OBJECT_ID('WORK.NS_BalPromoAnchor','U') IS NOT NULL DROP TABLE WORK.NS_BalPromoAnchor;
SELECT con_id, cli_id, out_rub, TSEGMENTNAME
INTO WORK.NS_BalPromoAnchor
FROM #bal_prom;

/* ---------- Циклы по договору (для определения окон) ---------- */
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

/* ---------- Календарь (дни) ---------- */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
CREATE TABLE #cal(d date NOT NULL PRIMARY KEY);
INSERT #cal VALUES (@Anchor);
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
BEGIN
  INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;
END;

/* ---------- Пул ключевых дат по каждому счёту ---------- */
IF OBJECT_ID('tempdb..#keys_raw') IS NOT NULL DROP TABLE #keys_raw;
CREATE TABLE #keys_raw(
  con_id       bigint      NOT NULL,
  cli_id       bigint      NOT NULL,
  TSEGMENTNAME nvarchar(40) NULL,
  cycle_no     int         NOT NULL,
  win_start    date        NOT NULL,
  promo_end    date        NOT NULL,
  base_day     date        NOT NULL,
  dt_key       date        NOT NULL,
  pri          tinyint     NOT NULL  -- 1=EOM, 2=1-е, 3=anchor
);
-- EOM
INSERT INTO #keys_raw
SELECT DISTINCT c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
       c.base_day AS dt_key, 1 AS pri
FROM #cycles c
WHERE c.base_day BETWEEN @Anchor AND @HorizonTo;
-- 1-е внутри окна
INSERT INTO #keys_raw
SELECT DISTINCT c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
       x.d AS dt_key, 2 AS pri
FROM #cycles c
JOIN #cal x ON x.d BETWEEN c.win_start AND c.base_day
WHERE DAY(x.d)=1;
-- 1-е нового окна
INSERT INTO #keys_raw
SELECT DISTINCT
  c.con_id, c.cli_id, c.TSEGMENTNAME,
  c.cycle_no+1,
  DATEADD(day,1,c.base_day),
  DATEADD(day,-1,EOMONTH(DATEADD(month,1,DATEADD(day,1,c.base_day)))),
  EOMONTH(DATEADD(month,1,DATEADD(day,1,c.base_day))),
  DATEADD(day,1,c.base_day),
  2
FROM #cycles c
WHERE DATEADD(day,1,c.base_day) BETWEEN @Anchor AND @HorizonTo;
-- Anchor (если внутри окна)
INSERT INTO #keys_raw
SELECT DISTINCT c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
       @Anchor, CASE WHEN @Anchor=c.base_day THEN 1 ELSE 3 END
FROM #cycles c
WHERE @Anchor BETWEEN c.win_start AND c.base_day;

IF OBJECT_ID('tempdb..#keys') IS NOT NULL DROP TABLE #keys;
;WITH pick AS (
  SELECT *, rn = ROW_NUMBER() OVER(
    PARTITION BY con_id, dt_key ORDER BY pri ASC, cycle_no DESC, con_id
  )
  FROM #keys_raw
)
SELECT con_id, cli_id, TSEGMENTNAME, cycle_no, win_start, promo_end, base_day, dt_key
INTO #keys
FROM pick
WHERE rn=1;

CREATE UNIQUE INDEX IX_keys ON #keys(con_id, dt_key);

/* ---------- Ставка на ключевую дату по счёту ---------- */
IF OBJECT_ID('tempdb..#rates_key') IS NOT NULL DROP TABLE #rates_key;
;WITH rk AS (
  SELECT
    x.con_id, x.cli_id, x.TSEGMENTNAME, x.cycle_no, x.win_start, x.promo_end, x.base_day, x.dt_key,
    rate_con =
    CASE
      WHEN x.dt_key = x.base_day THEN @BaseRate
      WHEN DAY(x.dt_key) = 1 THEN
           CASE
             WHEN @UseFixedPromoFromNextMonth=1
              AND x.dt_key>=@FixedPromoStart AND x.dt_key<@FixedPromoEnd
               THEN CASE x.TSEGMENTNAME WHEN N'ДЧБО' THEN @PromoRate_DChbo ELSE @PromoRate_Retail END
             WHEN @UseHardPromoSpread=1 AND x.dt_key>=@HardSpreadStart
               THEN CASE WHEN x.cycle_no=0 THEN bp.rate_anchor
                         ELSE CASE x.TSEGMENTNAME
                                WHEN N'ДЧБО' THEN @HardSpread_DChbo  + k.KEY_RATE
                                ELSE               @HardSpread_Retail+ k.KEY_RATE
                              END END
             ELSE CASE WHEN x.cycle_no=0 THEN bp.rate_anchor
                       ELSE CASE x.TSEGMENTNAME
                              WHEN N'ДЧБО' THEN @Spread_DChbo  + k.KEY_RATE
                              ELSE               @Spread_Retail+ k.KEY_RATE
                            END END
           END
      ELSE
           CASE
             WHEN @UseFixedPromoFromNextMonth=1
              AND x.dt_key>=@FixedPromoStart AND x.dt_key<@FixedPromoEnd
               THEN CASE x.TSEGMENTNAME WHEN N'ДЧБО' THEN @PromoRate_DChbo ELSE @PromoRate_Retail END
             WHEN @UseHardPromoSpread=1 AND x.dt_key>=@HardSpreadStart
               THEN CASE WHEN x.cycle_no=0 THEN bp.rate_anchor
                         ELSE CASE x.TSEGMENTNAME
                                WHEN N'ДЧБО' THEN @HardSpread_DChbo  + k.KEY_RATE
                                ELSE               @HardSpread_Retail+ k.KEY_RATE
                              END END
             ELSE CASE WHEN x.cycle_no=0 THEN bp.rate_anchor
                       ELSE CASE x.TSEGMENTNAME
                              WHEN N'ДЧБО' THEN @Spread_DChbo  + k.KEY_RATE
                              ELSE               @Spread_Retail+ k.KEY_RATE
                            END END
           END
    END
  FROM #keys x
  LEFT JOIN #key k       ON k.DT_REP = x.dt_key
  JOIN      #bal_prom bp ON bp.con_id = x.con_id
)
SELECT
  con_id, cli_id, TSEGMENTNAME, cycle_no, win_start, promo_end, base_day,
  dt_key, CAST(rate_con AS decimal(9,4)) AS rate_con
INTO #rates_key
FROM rk;

CREATE UNIQUE INDEX IX_rk ON #rates_key(con_id, dt_key);

/* ---------- Интервалы ставок (dt_from; dt_to) ---------- */
IF OBJECT_ID('WORK.NS_RateIntervals','U') IS NOT NULL DROP TABLE WORK.NS_RateIntervals;
;WITH ordered AS (
  SELECT con_id, cli_id, TSEGMENTNAME,
         dt_from = dt_key, rate_con,
         dt_to_next = LEAD(dt_key) OVER (PARTITION BY con_id ORDER BY dt_key)
  FROM #rates_key
),
bounds AS (
  SELECT con_id, cli_id, TSEGMENTNAME, rate_con,
         dt_from,
         dt_to = DATEADD(day,-1, ISNULL(dt_to_next, DATEADD(day,1,@HorizonTo)))
  FROM ordered
)
SELECT con_id, cli_id, TSEGMENTNAME, dt_from, dt_to, rate_con
INTO WORK.NS_RateIntervals
FROM bounds
WHERE dt_from <= @HorizonTo AND dt_to >= @Anchor;

CREATE UNIQUE CLUSTERED INDEX IX_NS_RI ON WORK.NS_RateIntervals(con_id, dt_from);
CREATE INDEX IX_NS_RI_cli ON WORK.NS_RateIntervals(cli_id, dt_from);

/* ============================================================
   ЭТАП 2: СОБЫТИЯ (EOM/1-е) → СЕГМЕНТЫ → ДНЕВНАЯ ЛЕНТА
   ============================================================ */

/* Ключевые даты для событий: EOM и 1-е */
IF OBJECT_ID('tempdb..#eom') IS NOT NULL DROP TABLE #eom;
SELECT d INTO #eom FROM #cal WHERE d = EOMONTH(d);

IF OBJECT_ID('tempdb..#d1') IS NOT NULL DROP TABLE #d1;
SELECT d INTO #d1 FROM #cal WHERE DAY(d)=1;

/* Якорный сплит: суммы по клиенту */
IF OBJECT_ID('tempdb..#anchor_bal') IS NOT NULL DROP TABLE #anchor_bal;
SELECT con_id=CAST(con_id AS bigint),
       cli_id=CAST(cli_id AS bigint),
       out_rub=CAST(out_rub AS decimal(20,2))
INTO #anchor_bal
FROM WORK.NS_BalPromoAnchor;

IF OBJECT_ID('tempdb..#cli_sum') IS NOT NULL DROP TABLE #cli_sum;
SELECT cli_id, out_rub_sum = SUM(out_rub)
INTO #cli_sum
FROM #anchor_bal
GROUP BY cli_id;
CREATE UNIQUE INDEX IX_clisum ON #cli_sum(cli_id);

/* Кандидаты на ключевые даты: все счета клиента */
IF OBJECT_ID('tempdb..#cand') IS NOT NULL DROP TABLE #cand;
SELECT c.cli_id, ab.con_id, k.d AS dt_rep
INTO #cand
FROM (SELECT DISTINCT cli_id FROM #anchor_bal) c
JOIN #anchor_bal ab ON ab.cli_id=c.cli_id
JOIN (SELECT d FROM #eom UNION ALL SELECT d FROM #d1) k ON 1=1;

CREATE INDEX IX_cand ON #cand(cli_id, dt_rep, con_id);

/* Ставка счета на ключевую дату из интервалов */
IF OBJECT_ID('tempdb..#cand_rates') IS NOT NULL DROP TABLE #cand_rates;
SELECT
  cand.cli_id, cand.con_id, cand.dt_rep,
  rate_on_date = ri.rate_con
INTO #cand_rates
FROM #cand cand
OUTER APPLY (
  SELECT TOP (1) r.rate_con
  FROM WORK.NS_RateIntervals r
  WHERE r.con_id=cand.con_id AND cand.dt_rep BETWEEN r.dt_from AND r.dt_to
  ORDER BY r.dt_from DESC
) ri;

CREATE INDEX IX_candr ON #cand_rates(cli_id, dt_rep, rate_on_date DESC, con_id);

/* События (winner per date) */
IF OBJECT_ID('WORK.NS_AllocEvents','U') IS NOT NULL DROP TABLE WORK.NS_AllocEvents;
;WITH ranked AS (
  SELECT cr.*,
         rn = ROW_NUMBER() OVER(
                PARTITION BY cr.cli_id, cr.dt_rep
                ORDER BY cr.rate_on_date DESC, cr.con_id
              )
  FROM #cand_rates cr
)
SELECT
  r.cli_id,
  r.con_id,
  r.dt_rep,
  out_rub = cs.out_rub_sum,
  reason  = CASE WHEN r.dt_rep IN (SELECT d FROM #eom) THEN 'EOM' ELSE 'D1' END
INTO WORK.NS_AllocEvents
FROM ranked r
JOIN #cli_sum cs ON cs.cli_id=r.cli_id
WHERE r.rn=1;

CREATE INDEX IX_aevt_cli_dt ON WORK.NS_AllocEvents(cli_id, dt_rep);
CREATE INDEX IX_aevt_con_dt ON WORK.NS_AllocEvents(con_id, dt_rep);

/* Сегменты по клиенту:
   - pre-first: [@Anchor ; first_evt-1] (держим якорный сплит)
   - затем для каждой записи events: [evt_i ; evt_{i+1}-1] (весь Σ на con_id) */
IF OBJECT_ID('tempdb..#first_evt') IS NOT NULL DROP TABLE #first_evt;
SELECT cli_id, first_evt = MIN(dt_rep)
INTO #first_evt
FROM WORK.NS_AllocEvents
GROUP BY cli_id;

IF OBJECT_ID('tempdb..#evt_seq') IS NOT NULL DROP TABLE #evt_seq;
;WITH e AS (
  SELECT cli_id, con_id, dt_rep,
         nxt = LEAD(dt_rep) OVER (PARTITION BY cli_id ORDER BY dt_rep)
  FROM WORK.NS_AllocEvents
)
SELECT cli_id, con_id,
       seg_start = dt_rep,
       seg_end   = DATEADD(day,-1, ISNULL(nxt, DATEADD(day,1,@HorizonTo)))
INTO #evt_seq
FROM e;

/* Дневная сетка */
IF OBJECT_ID('tempdb..#day') IS NOT NULL DROP TABLE #day;
SELECT d INTO #day FROM #cal;  -- уже есть @Anchor..@HorizonTo

/* Аллокации: pre-first (якорный сплит) */
IF OBJECT_ID('tempdb..#alloc_pre') IS NOT NULL DROP TABLE #alloc_pre;
SELECT
  d.d       AS dt_rep,
  ab.cli_id AS cli_id,
  ab.con_id AS con_id,
  ab.out_rub
INTO #alloc_pre
FROM #anchor_bal ab
JOIN #day d ON d.d >= @Anchor
LEFT JOIN #first_evt fe ON fe.cli_id = ab.cli_id
WHERE fe.first_evt IS NULL           -- клиент без событий: весь горизонт
   OR d.d < fe.first_evt;            -- до первого события

/* Аллокации: post (по сегментам winner) */
IF OBJECT_ID('tempdb..#alloc_post') IS NOT NULL DROP TABLE #alloc_post;
SELECT
  d.d       AS dt_rep,
  es.cli_id AS cli_id,
  es.con_id AS con_id,
  cs.out_rub_sum AS out_rub
INTO #alloc_post
FROM #evt_seq es
JOIN #day d           ON d.d BETWEEN es.seg_start AND es.seg_end
JOIN #cli_sum cs      ON cs.cli_id = es.cli_id;

/* Общая дневная аллокация: без пересечений сегментов → нет дублей */
IF OBJECT_ID('tempdb..#alloc_daily') IS NOT NULL DROP TABLE #alloc_daily;
SELECT * INTO #alloc_daily FROM #alloc_pre;
INSERT #alloc_daily(dt_rep, cli_id, con_id, out_rub)
SELECT dt_rep, cli_id, con_id, out_rub FROM #alloc_post;

CREATE INDEX IX_ad_con_dt ON #alloc_daily(con_id, dt_rep);

/* Ставка на день из интервалов (по каждому con_id) */
IF OBJECT_ID('tempdb..#rates_daily') IS NOT NULL DROP TABLE #rates_daily;
SELECT
  ad.dt_rep,
  ad.con_id,
  rate_con = ri.rate_con
INTO #rates_daily
FROM #alloc_daily ad
OUTER APPLY (
  SELECT TOP (1) r.rate_con
  FROM WORK.NS_RateIntervals r
  WHERE r.con_id=ad.con_id AND ad.dt_rep BETWEEN r.dt_from AND r.dt_to
  ORDER BY r.dt_from DESC
) ri;

CREATE INDEX IX_rd_con_dt ON #rates_daily(con_id, dt_rep);

/* Итоговая дневная агрегированная лента */
IF OBJECT_ID('WORK.Forecast_NS_Promo','U') IS NOT NULL DROP TABLE WORK.Forecast_NS_Promo;
SELECT
  ad.dt_rep,
  out_rub_total = SUM(ad.out_rub),
  rate_avg      = SUM(ad.out_rub * CAST(rd.rate_con AS decimal(9,4))) / NULLIF(SUM(ad.out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM #alloc_daily ad
JOIN #rates_daily rd
  ON rd.con_id = ad.con_id AND rd.dt_rep = ad.dt_rep
GROUP BY ad.dt_rep;

/* ---------------- Контрольки ---------------- */
PRINT N'=== Anchor split (TOP 20) ===';
SELECT TOP (20) * FROM WORK.NS_BalPromoAnchor ORDER BY cli_id, con_id;

PRINT N'=== Rate intervals (TOP 20) ===';
SELECT TOP (20) * FROM WORK.NS_RateIntervals ORDER BY con_id, dt_from;

PRINT N'=== Events (TOP 50) ===';
SELECT TOP (50) * FROM WORK.NS_AllocEvents ORDER BY cli_id, dt_rep;

PRINT N'=== Daily aggregate (TOP 200) ===';
SELECT TOP (200) f.dt_rep, f.out_rub_total,
       diff_vs_anchor = CAST(f.out_rub_total - @AnchorTotal AS decimal(20,2)),
       f.rate_avg
FROM WORK.Forecast_NS_Promo f
ORDER BY f.dt_rep;

Что критично исправлено/усилено:
	•	Отвязал формирование аллокаций от промежуточной ленты #daily_pre: теперь только сегменты без пересечений, значит нет дублей объёмов.
	•	Победитель определяется только на EOM/1-е, по ставке аккаунта на дату из NS_RateIntervals.
	•	До первого события — ровно якорный сплит (несколько счетов с их объёмами).
	•	После события — весь Σ клиента строго на один con_id-победитель до следующего события.
	•	Ставка на любой день — из интервалов по счёту, мы никогда не подменяем ставки проигравших счетов.
	•	Все JOIN’ы квалифицированы, индексы на узких местах добавлены, нет sys.*.

Если что-то поведёт себя не так — дай конкретные cli_id / con_id / dt_rep, я точечно разберу на этой схеме.
