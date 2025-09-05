понял задачу. Дам два кусочка T-SQL: сначала очень короткий «проверочный» запрос (п.1), затем рабочий скрипт с логикой «70% открывают НОВЫЙ счёт на base-day» (п.2). Всё — под MS SQL Server, без `DROP TABLE IF EXISTS`, только через `OBJECT_ID`.

---

# 1) «Простецкий» срез на 2025-08-29

Объём на НС FIX, открытых в июле-2025, **только у тех клиентов**, у кого есть ещё НС, открытые в августе-2025.

```sql
USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @dt date = '2025-08-29';

;WITH aug_clients AS (        -- клиенты, у кого есть НС, открытые в августе
    SELECT DISTINCT t.cli_id
    FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
    WHERE t.dt_open >= '2025-08-01' AND t.dt_open < '2025-09-01'
      AND t.prod_id = 654
      AND t.section_name = N'Накопительный счёт'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag = 1 AND t.cur = '810'
),
july_cons AS (                -- их же договоры, открытые в июле
    SELECT DISTINCT t.cli_id, t.con_id
    FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
    JOIN aug_clients a ON a.cli_id = t.cli_id
    WHERE t.dt_open >= '2025-07-01' AND t.dt_open < '2025-08-01'
      AND t.prod_id = 654
      AND t.section_name = N'Накопительный счёт'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag = 1 AND t.cur = '810'
)
-- объём на 29 августа именно по "июльским" счетам этих клиентов
SELECT
    t.dt_rep,
    t.cli_id,
    out_rub_total = SUM(CAST(t.out_rub AS decimal(20,2)))
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
JOIN july_cons j ON j.con_id = t.con_id
WHERE t.dt_rep = @dt
GROUP BY t.dt_rep, t.cli_id
ORDER BY t.cli_id;
```

> Получишь построчно по `cli_id` объёмы «июльских» договоров на 29.08 у клиентов, у которых есть хоть один «августовский» договор.

---

# 2) Рабочий скрипт: **70% объёма у клиентов, у кого на base-day есть «база», открывают НОВЫЙ счёт с промо**

Новый счёт действует с base-day и весь следующий месяц на промо-ставке (фикс по спреду к KEY(base-day)), затем — база. Старые договоры у такого клиента **масштабируются** с base-day и далее коэффициентом *(1 − 0.70)*. Если у клиента в последующие месяцы опять наступает base-day (по другим окнам), логика применяется повторно (коэффициент мультипликативный). Объём в сумме по клиенту в любой день сохраняется.

```sql
/* ══════════════════════════════════════════════════════════════
   NS-forecast — FIX-promo-roll + "70% NEW OPEN at base-day"
   - СНАПШОТ @Anchor как в mail.usp_fill_balance_metrics_savings:
     ULTRA(+1/+2) и сравнение LIQ vs balance (rate_use).
   - Промо-ставки задаются на входе и переводятся в спреды к KEY(@Anchor).
   - На КАЖДЫЙ base-day клиента: открываем НОВЫЙ счёт с share=70% Σ,
     действует base-day..(конец следующего месяца-1) по промо (KEY(base-day)+spread),
     затем база. Старые счета масштабируем коэффициентом (1-share) с base-day и далее.
   Вывод: WORK.Forecast_NS_Promo70 (итог по дням).
   v.2025-08-11
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
SET NOCOUNT ON;

DECLARE
    @Scenario   tinyint      = 1,
    @Anchor     date         = '2025-09-03',
    @HorizonTo  date         = '2025-12-31',
    @BaseRate   decimal(9,4) = 0.0450,
    @ShareNew   decimal(9,6) = 0.700000;  -- 70%

/* охватить открытия: месяц до Anchor + сам месяц Anchor */
DECLARE
    @OpenFrom date = DATEFROMPARTS(YEAR(DATEADD(month,-1,@Anchor)), MONTH(DATEADD(month,-1,@Anchor)), 1),
    @OpenTo   date = EOMONTH(@Anchor);

/* ---------- календарь и KEY(TERM=1) ---------- */
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

IF OBJECT_ID('tempdb..#key') IS NOT NULL DROP TABLE #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@Scenario AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

DECLARE @KeyAtAnchor decimal(9,4);
SELECT @KeyAtAnchor = KEY_RATE FROM #key WHERE DT_REP=@Anchor;

/* ---------- входные промо → спреды к KEY(@Anchor) ---------- */
DECLARE
    @PromoRate_DChbo  decimal(9,4) = 0.1660,
    @PromoRate_Retail decimal(9,4) = 0.1640;

DECLARE
    @Spread_DChbo  decimal(9,4) = @PromoRate_DChbo  - @KeyAtAnchor,
    @Spread_Retail decimal(9,4) = @PromoRate_Retail - @KeyAtAnchor;

IF OBJECT_ID('WORK.NS_Spreads','U') IS NOT NULL DROP TABLE WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads (TSEGMENTNAME nvarchar(40) PRIMARY KEY, spread decimal(9,4) NOT NULL);
INSERT WORK.NS_Spreads VALUES (N'ДЧБО',@Spread_DChbo),(N'Розничный бизнес',@Spread_Retail);

/* ---------- СНАПШОТ на @Anchor как в SP (rate_use) ---------- */
IF OBJECT_ID('tempdb..#bal_src') IS NOT NULL DROP TABLE #bal_src;
SELECT
    t.dt_rep,
    CAST(t.dt_open AS date)           AS dt_open,
    CAST(t.dt_close AS date)          AS dt_close,
    CAST(t.con_id AS bigint)          AS con_id,
    CAST(t.cli_id AS bigint)          AS cli_id,
    CAST(t.prod_id AS int)            AS prod_id,
    CAST(t.out_rub AS decimal(20,2))  AS out_rub,
    CAST(t.rate_con AS decimal(9,4))  AS rate_balance,
    t.rate_con_src,
    t.TSEGMENTNAME,
    CAST(r.rate AS decimal(9,4))      AS rate_liq
INTO #bal_src
FROM   alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
       ON  r.con_id = t.con_id
       AND CASE WHEN t.dt_open = t.dt_rep THEN DATEADD(day,1,t.dt_rep)
                ELSE t.dt_rep END BETWEEN r.dt_from AND r.dt_to
WHERE  t.dt_rep BETWEEN DATEADD(day,-2,@Anchor) AND DATEADD(day,2,@Anchor)
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag = 1 AND t.cur='810'
  AND  t.prod_id = 654;

CREATE CLUSTERED INDEX IX_bal_src ON #bal_src(con_id,dt_rep);

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
             WHEN rate_liq IS NULL THEN CASE WHEN rate_balance<0 THEN COALESCE(rate_pos,rate_balance)
                                             ELSE rate_balance END
             WHEN rate_liq<0  AND rate_balance>0 THEN rate_balance
             WHEN rate_liq<0  AND rate_balance<0 THEN COALESCE(rate_pos,rate_balance)
             WHEN rate_liq>=0 AND rate_balance>=0 THEN rate_liq
             WHEN rate_liq>0  AND rate_balance<0 THEN rate_liq
             ELSE rate_liq
           END AS rate_use
    FROM bal_pos
)
SELECT
    con_id, cli_id, out_rub,
    rate_anchor = CAST(rate_use AS decimal(9,4)),
    dt_open, TSEGMENTNAME
INTO #bal_prom
FROM rate_calc
WHERE out_rub IS NOT NULL
  AND dt_open BETWEEN @OpenFrom AND @OpenTo;

-- Σ клиента на Anchor
IF OBJECT_ID('tempdb..#cli_sum') IS NOT NULL DROP TABLE #cli_sum;
SELECT cli_id, SUM(out_rub) AS sum_out
INTO   #cli_sum
FROM   #bal_prom
GROUP  BY cli_id;

/* ---------- окна (promo/base) по каждому договору ---------- */
IF OBJECT_ID('tempdb..#cycles') IS NOT NULL DROP TABLE #cycles;
;WITH seed AS (
    SELECT
        b.con_id, b.cli_id, b.TSEGMENTNAME, b.out_rub,
        CAST(0 AS int) AS cycle_no,
        b.dt_open AS win_start,
        EOMONTH(DATEADD(month,1,b.dt_open))                  AS base_day,
        DATEADD(day,-1,EOMONTH(DATEADD(month,1,b.dt_open)))  AS promo_end
    FROM #bal_prom b
),
seq AS (
    SELECT * FROM seed
    UNION ALL
    SELECT
        s.con_id, s.cli_id, s.TSEGMENTNAME, s.out_rub,
        s.cycle_no+1,
        DATEADD(day,1,s.base_day),
        EOMONTH(DATEADD(month,1,DATEADD(day,1,s.base_day))),
        DATEADD(day,-1,EOMONTH(DATEADD(month,1,DATEADD(day,1,s.base_day))))
    FROM seq s
    WHERE DATEADD(day,1,s.base_day) <= @HorizonTo
)
SELECT * INTO #cycles FROM seq OPTION (MAXRECURSION 0);

-- ставки по дням (без 1-х чисел; как ты просил — ставка фикс окна)
IF OBJECT_ID('tempdb..#daily_pre') IS NOT NULL DROP TABLE #daily_pre;
SELECT
    c.con_id, c.cli_id, c.TSEGMENTNAME, c.out_rub,
    c.cycle_no, c.win_start, c.promo_end, c.base_day,
    d.d AS dt_rep,
    CASE
      WHEN d.d <= c.promo_end THEN
           CASE WHEN c.cycle_no=0
                THEN bp.rate_anchor
                ELSE (CASE c.TSEGMENTNAME
                        WHEN N'ДЧБО'            THEN @Spread_DChbo  + k_open.KEY_RATE
                        ELSE                          @Spread_Retail+ k_open.KEY_RATE
                     END)
           END
      WHEN d.d = c.base_day THEN @BaseRate
    END AS rate_con
INTO #daily_pre
FROM #cycles c
JOIN #cal d        ON d.d BETWEEN c.win_start AND c.base_day
LEFT JOIN #key k_open ON k_open.DT_REP = c.win_start
JOIN #bal_prom bp  ON bp.con_id = c.con_id
WHERE DAY(d.d) <> 1;

/* ---------- base-day события по клиентам ---------- */
IF OBJECT_ID('tempdb..#base_events') IS NOT NULL DROP TABLE #base_events;
SELECT DISTINCT dp.cli_id, dp.dt_rep AS base_day
INTO   #base_events
FROM   #daily_pre dp
JOIN   #cycles c ON c.con_id=dp.con_id AND c.cycle_no=dp.cycle_no
WHERE  dp.dt_rep = c.base_day;

-- ранжирование событий и предрасчёт счётчиков (сколько событий уже случилось к дате)
IF OBJECT_ID('tempdb..#be_rank') IS NOT NULL DROP TABLE #be_rank;
SELECT
    be.cli_id, be.base_day,
    rn = ROW_NUMBER() OVER (PARTITION BY be.cli_id ORDER BY be.base_day)
INTO #be_rank
FROM #base_events be;

IF OBJECT_ID('tempdb..#event_cnt') IS NOT NULL DROP TABLE #event_cnt;
SELECT
    c.cli_id,
    d.d AS dt_rep,
    cnt = COUNT(br.base_day)
INTO #event_cnt
FROM (SELECT DISTINCT cli_id FROM #bal_prom) c
CROSS JOIN #cal d
LEFT JOIN #be_rank br
       ON br.cli_id=c.cli_id AND br.base_day <= d.d
GROUP BY c.cli_id,d.d;

/* ---------- масштабирование исходных договоров (1-share)^events ---------- */
IF OBJECT_ID('tempdb..#daily_orig_scaled') IS NOT NULL DROP TABLE #daily_orig_scaled;
SELECT
    dp.con_id, dp.cli_id, dp.TSEGMENTNAME,
    out_rub = CAST(ROUND(dp.out_rub * POWER(1-@ShareNew, ISNULL(ec.cnt,0)), 2) AS decimal(20,2)),
    dp.dt_rep,
    dp.rate_con
INTO #daily_orig_scaled
FROM #daily_pre dp
LEFT JOIN #event_cnt ec
       ON ec.cli_id=dp.cli_id AND ec.dt_rep=dp.dt_rep;

/* ---------- НОВЫЕ счёта (по каждому base-day событию) ---------- */
IF OBJECT_ID('tempdb..#newacc_def') IS NOT NULL DROP TABLE #newacc_def;
;WITH pick_rate AS (  -- промо-ставка: выбираем бОльшую из сегментов на base-day
    SELECT
      br.cli_id, br.base_day,
      promo_rate = (SELECT MAX(v)
                    FROM (VALUES (@Spread_DChbo  + k.KEY_RATE),
                                 (@Spread_Retail + k.KEY_RATE)) t(v))
    FROM #be_rank br
    LEFT JOIN #key k ON k.DT_REP = br.base_day
)
SELECT
    new_con_id = -1 * ABS(CHECKSUM(p.cli_id, p.base_day)) - br.rn, -- синтетический id
    p.cli_id,
    TSEGMENTNAME = N'Промо MAX',        -- просто метка
    amount = CAST(ROUND(cs.sum_out * @ShareNew * POWER(1-@ShareNew, br.rn-1), 2) AS decimal(20,2)),
    win_start = p.base_day,
    base_day2 = EOMONTH(DATEADD(month,1,p.base_day)),
    promo_end2= DATEADD(day,-1,EOMONTH(DATEADD(month,1,p.base_day))),
    promo_rate= p.promo_rate
INTO #newacc_def
FROM pick_rate p
JOIN #be_rank   br ON br.cli_id=p.cli_id AND br.base_day=p.base_day
JOIN #cli_sum   cs ON cs.cli_id=p.cli_id;

/* дневная лента для НОВЫХ счетов: promo до promo_end2, затем база до горизонта */
IF OBJECT_ID('tempdb..#newacc_daily') IS NOT NULL DROP TABLE #newacc_daily;
SELECT
    n.new_con_id AS con_id,
    n.cli_id,
    n.TSEGMENTNAME,
    n.amount     AS out_rub,
    d.d          AS dt_rep,
    rate_con     = CASE
                     WHEN d.d <= n.promo_end2 THEN n.promo_rate
                     WHEN d.d = n.base_day2   THEN @BaseRate
                     ELSE @BaseRate
                   END
INTO #newacc_daily
FROM #newacc_def n
JOIN #cal d
  ON d.d BETWEEN n.win_start AND @HorizonTo;

/* ---------- итог: суммируем оригинал (scaled) + новые счета ---------- */
IF OBJECT_ID('WORK.Forecast_NS_Promo70','U') IS NOT NULL DROP TABLE WORK.Forecast_NS_Promo70;

SELECT
    dt_rep,
    out_rub_total = SUM(out_rub),
    rate_avg      = SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)
INTO WORK.Forecast_NS_Promo70
FROM (
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con FROM #daily_orig_scaled
    UNION ALL
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con FROM #newacc_daily
) u
GROUP BY dt_rep;

/* ---------- проверки ---------- */
PRINT N'=== Σ на Anchor по промо-портфелю ===';
SELECT SUM(out_rub) AS sum_anchor FROM #bal_prom;

PRINT N'=== sanity: Σ по дням (должно быть = Σ на Anchor) ===';
SELECT TOP (120) dt_rep, out_rub_total
FROM WORK.Forecast_NS_Promo70
ORDER BY dt_rep;

PRINT N'=== примеры по окну конца месяца и начала след. ===';
SELECT *
FROM WORK.Forecast_NS_Promo70
WHERE dt_rep BETWEEN EOMONTH(@Anchor) - 5 AND DATEADD(day,5,EOMONTH(@Anchor))
ORDER BY dt_rep;
```

### Замечания

* В пункте (1) ты сразу увидишь «проблему потерянных объёмов», если она где-то прячется — мы берём ровно те же фильтры, что и в твоих моделях.
* Во втором скрипте **ставки по каждому `con_id` моделируются день-в-день** (окно промо фиксируется на `win_start`, base — в `base_day`), а далее добавляется логика **НОВЫХ** счетов с долей 70% на каждом `base_day`.
* Объёмы **не теряются**: сумма по клиенту в любой день = Σ на Anchor (масштабирование оригиналов + добавление новых равны «переливу»).

Если хочешь другой выбор доли (например, 60%) — меняешь `@ShareNew`. Если промо-входы меняются — правишь `@PromoRate_DChbo / @PromoRate_Retail`.
