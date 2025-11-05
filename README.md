/* ============================================================
   PART 1: NS (prod_id = 654) — RATE INTERVALS BY ACCOUNT
   Результат:
     • WORK.NS_BalPromoAnchor(con_id, cli_id, out_rub, tsegmentname)
     • WORK.NS_RateIntervals(con_id, cli_id, tsegmentname,
                             dt_from, dt_to, rate_con)
   Ключевые даты: @Anchor, все EOM и все 1-е в диапазоне.
   ============================================================ */

USE ALM_TEST;
SET NOCOUNT ON;

/* ---------- параметры ---------- */
DECLARE
    @Scenario   tinyint      = 1,
    @Anchor     date         = '2025-11-03',
    @HorizonTo  date         = '2026-03-31',
    @BaseRate   decimal(9,4) = 0.0450,  -- базовая ставка в base_day/EOM
    @UseFixedPromoFromNextMonth bit = 0,
    @UseHardPromoSpread bit         = 0;

/* окно фикс-промо: 1 месяц после якоря */
DECLARE @FixedPromoStart date = DATEADD(day,1,EOMONTH(@Anchor));
DECLARE @FixedPromoEnd   date = DATEADD(month,1,@FixedPromoStart);
DECLARE @HardSpreadStart date = @Anchor;

/* ---------- ключевая (TERM=1) ---------- */
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

/* ---------- «живые» договоры к якорю ---------- */
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
       AND CASE WHEN t.dt_open=t.dt_rep THEN DATEADD(day,1,t.dt_rep) ELSE t.dt_rep END
           BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day,2,@Anchor)
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.cur          = '810'
  AND  t.prod_id      = 654;

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src(con_id, dt_rep);

/* ---------- ставка на якоре по каждому договору ---------- */
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

CREATE UNIQUE CLUSTERED INDEX IX_bal_prom ON #bal_prom(con_id);

/* ---------- нулевые договоры к якорю ---------- */
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

/* ---------- сохраняем сплит якоря ---------- */
IF OBJECT_ID('WORK.NS_BalPromoAnchor','U') IS NOT NULL DROP TABLE WORK.NS_BalPromoAnchor;
SELECT con_id, cli_id, out_rub, TSEGMENTNAME
INTO WORK.NS_BalPromoAnchor
FROM #bal_prom;

/* ---------- строим циклы окна по каждому договору ---------- */
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

/* ---------- календарь ключевых дат через WHILE ---------- */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
CREATE TABLE #cal (d date NOT NULL PRIMARY KEY);
INSERT #cal VALUES (@Anchor);
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
BEGIN
  INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;
END

/* ---------- создаём пул ключевых дат (#keys_raw) ЗАРАНЕЕ ---------- */
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
  pri          tinyint     NOT NULL  -- 1=EOM/base_day, 2=1st, 3=anchor/other
);

/* --- EOM (base_day) --- */
INSERT INTO #keys_raw(con_id,cli_id,TSEGMENTNAME,cycle_no,win_start,promo_end,base_day,dt_key,pri)
SELECT DISTINCT
  c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
  c.base_day AS dt_key, 1 AS pri
FROM #cycles c
WHERE c.base_day BETWEEN @Anchor AND @HorizonTo;

/* --- 1-е внутри окна --- */
INSERT INTO #keys_raw(con_id,cli_id,TSEGMENTNAME,cycle_no,win_start,promo_end,base_day,dt_key,pri)
SELECT DISTINCT
  c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
  x.d AS dt_key, 2 AS pri
FROM #cycles c
JOIN #cal x ON x.d BETWEEN c.win_start AND c.base_day
WHERE DAY(x.d)=1;

/* --- 1-е нового окна (base_day+1) --- */
INSERT INTO #keys_raw(con_id,cli_id,TSEGMENTNAME,cycle_no,win_start,promo_end,base_day,dt_key,pri)
SELECT DISTINCT
  c.con_id, c.cli_id, c.TSEGMENTNAME,
  c.cycle_no+1 AS cycle_no,
  DATEADD(day,1,c.base_day) AS win_start,
  DATEADD(day,-1,EOMONTH(DATEADD(month,1,DATEADD(day,1,c.base_day)))) AS promo_end,
  EOMONTH(DATEADD(month,1,DATEADD(day,1,c.base_day))) AS base_day,
  DATEADD(day,1,c.base_day) AS dt_key,
  2 AS pri
FROM #cycles c
WHERE DATEADD(day,1,c.base_day) BETWEEN @Anchor AND @HorizonTo;

/* --- якорь, если он попадает внутри окна --- */
INSERT INTO #keys_raw(con_id,cli_id,TSEGMENTNAME,cycle_no,win_start,promo_end,base_day,dt_key,pri)
SELECT DISTINCT
  c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
  @Anchor AS dt_key,
  CASE WHEN @Anchor=c.base_day THEN 1 ELSE 3 END AS pri
FROM #cycles c
WHERE @Anchor BETWEEN c.win_start AND c.base_day;

/* ---------- дедуп по (con_id, dt_key) по приоритету ---------- */
IF OBJECT_ID('tempdb..#keys') IS NOT NULL DROP TABLE #keys;
;WITH pick AS (
  SELECT *,
         rn = ROW_NUMBER() OVER(
                PARTITION BY con_id, dt_key
                ORDER BY pri ASC, cycle_no DESC, con_id
              )
  FROM #keys_raw
)
SELECT con_id, cli_id, TSEGMENTNAME, cycle_no, win_start, promo_end, base_day, dt_key
INTO #keys
FROM pick
WHERE rn=1;

CREATE UNIQUE INDEX IX_keys_unique ON #keys(con_id, dt_key);

/* ---------- вычисляем ставку на ключевую дату ---------- */
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
  FROM #keys x
  LEFT JOIN #key k       ON k.DT_REP = x.dt_key
  JOIN      #bal_prom bp ON bp.con_id = x.con_id
)
SELECT
  con_id, cli_id, TSEGMENTNAME, cycle_no, win_start, promo_end, base_day,
  dt_key, CAST(rate_con AS decimal(9,4)) AS rate_con
INTO #rates_key
FROM rk;

CREATE UNIQUE INDEX IX_rk_unique ON #rates_key(con_id, dt_key);

/* ---------- превращаем ключевые точки в интервалы ---------- */
IF OBJECT_ID('WORK.NS_RateIntervals','U') IS NOT NULL DROP TABLE WORK.NS_RateIntervals;

;WITH ordered AS (
  SELECT
    con_id, cli_id, TSEGMENTNAME,
    dt_from = dt_key,
    rate_con,
    dt_to_next = LEAD(dt_key) OVER (PARTITION BY con_id ORDER BY dt_key)
  FROM #rates_key
),
bounds AS (
  SELECT
    con_id, cli_id, TSEGMENTNAME, rate_con,
    dt_from,
    dt_to = DATEADD(day,-1,ISNULL(dt_to_next, DATEADD(day,1,@HorizonTo)))
  FROM ordered
)
SELECT
  con_id, cli_id, TSEGMENTNAME,
  dt_from, dt_to, rate_con
INTO WORK.NS_RateIntervals
FROM bounds
WHERE dt_from <= @HorizonTo
  AND dt_to   >= @Anchor;

CREATE UNIQUE CLUSTERED INDEX IX_NS_RateIntervals ON WORK.NS_RateIntervals(con_id, dt_from);
CREATE INDEX IX_NS_RateIntervals_cli ON WORK.NS_RateIntervals(cli_id, dt_from);
