/* ============================================================
   PART 1: NS (prod_id = 654) — RATE INTERVALS BY ACCOUNT (FIX)
   Пишет:
     • WORK.NS_BalPromoAnchor(con_id, cli_id, out_rub, tsegmentname)
     • WORK.NS_RateIntervals(con_id, cli_id, tsegmentname, dt_from, dt_to, rate_con)
   Ключевые даты по каждому ЦИКЛУ договора c дедупликацией:
     - @Anchor (если попадает в окно цикла)
     - все EOM внутри окна
     - 1-е числа внутри окна
     - 1-е НОВОГО окна (= base_day + 1), если ≤ @HorizonTo
   При коллизии одного и того же dt_key берём строго один вариант:
     pri=1: base_day (=EOM базового месяца)  >  pri=2: 1-е  >  pri=3: прочие (якорь)
   Между ключевыми датами — постоянная ставка (интервалы).
   ============================================================ */

USE ALM_TEST;
SET NOCOUNT ON;

/* ---------- параметры ---------- */
DECLARE
    @Scenario   tinyint      = 1,
    @Anchor     date         = '2025-11-03',
    @HorizonTo  date         = '2026-03-31',
    @BaseRate   decimal(9,4) = 0.0450,  -- базовая ставка в base_day/EOM
    @UseFixedPromoFromNextMonth bit = 0, -- фикс-промо только в месяце, следующем за якорем
    @UseHardPromoSpread bit         = 0; -- жёсткие спреды вместо расчётных

/* окно фикс-промо: ровно месяц после якоря */
DECLARE @FixedPromoStart date = DATEADD(day,1,EOMONTH(@Anchor));   -- 1-е след. месяца
DECLARE @FixedPromoEnd   date = DATEADD(month,1,@FixedPromoStart); -- 1-е через месяц
DECLARE @HardSpreadStart date = @Anchor;

/* ---------- ключевая ставка ЦБ (TERM=1) ---------- */
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

/* ---------- сохраняем компактный сплит якоря ---------- */
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

/* ---------- календарь (WHILE), без рекурсий ---------- */
IF OBJECT_ID('tempdb..#cal_small') IS NOT NULL DROP TABLE #cal_small;
CREATE TABLE #cal_small (d date PRIMARY KEY);
INSERT #cal_small VALUES (@Anchor);
WHILE (SELECT MAX(d) FROM #cal_small) < @HorizonTo
BEGIN
  INSERT #cal_small
  SELECT DATEADD(day,1,MAX(d)) FROM #cal_small;
END

/* ---------- сырой пул ключевых дат по циклам ---------- */
IF OBJECT_ID('tempdb..#keys_raw') IS NOT NULL DROP TABLE #keys_raw;
-- EOM внутри окна
INSERT INTO #keys_raw
SELECT DISTINCT
  c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
  dt_key = EOMONTH(x.d),
  pri    = 1  -- приоритет base_day/EOM
FROM #cycles c
JOIN #cal_small x ON x.d BETWEEN c.win_start AND c.base_day
WHERE EOMONTH(x.d) = c.base_day;

-- 1-е числа внутри окна
INSERT INTO #keys_raw
SELECT DISTINCT
  c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
  dt_key = x.d,
  pri    = 2
FROM #cycles c
JOIN #cal_small x ON x.d BETWEEN c.win_start AND c.base_day
WHERE DAY(x.d)=1;

-- 1-е нового окна
INSERT INTO #keys_raw
SELECT DISTINCT
  c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
  dt_key = DATEADD(day,1,c.base_day),
  pri    = 2  -- тоже 1-е число
FROM #cycles c
WHERE DATEADD(day,1,c.base_day) <= @HorizonTo;

-- якорь, если попадает в окно
INSERT INTO #keys_raw
SELECT DISTINCT
  c.con_id, c.cli_id, c.TSEGMENTNAME, c.cycle_no, c.win_start, c.promo_end, c.base_day,
  dt_key = @Anchor,
  pri    = CASE WHEN @Anchor = c.base_day THEN 1 ELSE 3 END  -- если якорь совпал с base_day — pri=1
FROM #cycles c
WHERE @Anchor BETWEEN c.win_start AND c.base_day;

-- объявление таблицы #keys_raw (нужно после первого INSERT SELECT)
IF @@ROWCOUNT >= 0 AND OBJECT_ID('tempdb..#keys_raw') IS NULL
BEGIN
    CREATE TABLE #keys_raw(
      con_id bigint, cli_id bigint, TSEGMENTNAME nvarchar(40),
      cycle_no int, win_start date, promo_end date, base_day date,
      dt_key date, pri tinyint
    );
END

/* ---------- выбираем ровно ОДНУ строку на (con_id, dt_key) ---------- */
IF OBJECT_ID('tempdb..#keys') IS NOT NULL DROP TABLE #keys;
;WITH picked AS (
  SELECT *,
         rn = ROW_NUMBER() OVER(
               PARTITION BY con_id, dt_key
               ORDER BY pri ASC, cycle_no DESC, con_id
         )
  FROM #keys_raw
)
SELECT
  con_id, cli_id, TSEGMENTNAME, cycle_no, win_start, promo_end, base_day, dt_key
INTO #keys
FROM picked
WHERE rn = 1;

CREATE UNIQUE INDEX IX_keys_unique ON #keys(con_id, dt_key);

/* ---------- ставка на ключевую дату с приоритетами ---------- */
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
           /* «прочие» (например, якорь, если не base_day и не 1-е) */
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

/* ---------- формирование ИНТЕРВАЛОВ ставок ---------- */
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
    dt_to = DATEADD(day,-1,ISNULL(dt_to_next, DATEADD(day,1,@HorizonTo))) -- последний тянем до HorizonTo
  FROM ordered
)
SELECT
  con_id, cli_id, TSEGMENTNAME,
  dt_from,
  dt_to,
  rate_con
INTO WORK.NS_RateIntervals
FROM bounds
WHERE dt_from <= @HorizonTo
  AND dt_to   >= @Anchor;

CREATE UNIQUE CLUSTERED INDEX IX_NS_RateIntervals ON WORK.NS_RateIntervals(con_id, dt_from);
CREATE INDEX IX_NS_RateIntervals_cli ON WORK.NS_RateIntervals(cli_id, dt_from);
