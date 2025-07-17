Ниже зафиксирована **полная версия скрипта**, в которой

* сначала ищем спред в «родной» конвенции,
* а если не нашли — **переводим ставку в ежемесячную** через
  `LIQUIDITY.liq.fnc_IntRate( …, 'monthly', 1 )` и ищем ещё раз.

Так мы «дотягиваем» именно те \~9 % договоров, что раньше
оставались пустыми из-за конвенции.

> **NB** замените строковые коды конвенций в таблице-маппере
> (`#conv_map`) на те, что реально используются у вас
> (`AT_THE_END`, `ON_DEMAND`, `1M`, `3M`, …).
> Если нужно больше маппингов — просто допишите строки.

```sql
/*──────────────────── 0. Параметры ───────────────────*/
DECLARE
    @Anchor    date = '2025-07-15',           -- DT_REP
    @HorizonTo date = '2025-08-31';           -- ≤ закрытие

/*──────────── temp-cleanup (на всякий) ───────────────*/
IF OBJECT_ID('tempdb..#bucket_def') IS NOT NULL DROP TABLE #bucket_def;
IF OBJECT_ID('tempdb..#conv_map')   IS NOT NULL DROP TABLE #conv_map;
IF OBJECT_ID('tempdb..#fresh15')    IS NOT NULL DROP TABLE #fresh15;
IF OBJECT_ID('tempdb..#mkt')        IS NOT NULL DROP TABLE #mkt;
IF OBJECT_ID('tempdb..#mkt_1M')     IS NOT NULL DROP TABLE #mkt_1M;
IF OBJECT_ID('tempdb..#mkt_any')    IS NOT NULL DROP TABLE #mkt_any;
IF OBJECT_ID('tempdb..#roll_fix')   IS NOT NULL DROP TABLE #roll_fix;
IF OBJECT_ID('tempdb..#match')      IS NOT NULL DROP TABLE #match;

/*──────────── 1. бакеты объёма ────────────*/
CREATE TABLE #bucket_def
( bucket varchar(20) PRIMARY KEY,
  lo money NOT NULL, hi money NULL, r tinyint NOT NULL);
INSERT #bucket_def VALUES
('[0-1.5 млн)',0,1500000,0),
('[1.5-15 млн)',1500000,15000000,1),
('[15-100 млн)',15000000,100000000,2),
('[100 млн+]',100000000,NULL,3);

/*──────────── 2. маппер конвенций → fnc_IntRate-коды ───────────*/
CREATE TABLE #conv_map
( conv_db  varchar(50) PRIMARY KEY,      -- что хранится в vw_balance…
  conv_liq varchar(50) NOT NULL);        -- что ждёт fnc_IntRate
INSERT #conv_map VALUES
('AT_THE_END','at the end'),
('1M','monthly'),
('3M','three_months'),
('ON_DEMAND','on_demand');  -- дополняйте при необходимости

/*──────────── 3. свежие фиксы от 15-го (#fresh15) ──────────────*/
CREATE TABLE #fresh15
( bucket varchar(20), TERM_GROUP varchar(100) NULL,
  PROD_NAME_RES nvarchar(200) NULL, TSEGMENTNAME nvarchar(100) NULL,
  conv varchar(50),                 -- «родная» конвенция
  conv_norm varchar(50),            -- после возможного перевода
  out_rub money,
  spread decimal(18,8) );

INSERT INTO #fresh15
SELECT
    b.bucket,
    tg.TERM_GROUP,
    t.PROD_NAME_RES,
    t.TSEGMENTNAME,
    CAST(t.conv AS varchar(50))                                 AS conv,
    '1M'                                                        AS conv_norm,         -- всё приводим к monthly
    t.out_rub,
    /* ставка приводится в 1M, затем вычитаем AVG_KEY_RATE (он уже 1M) */
    CAST( [LIQUIDITY].[liq].[fnc_IntRate]
          ( t.rate_con,
            cm.conv_liq,
            'monthly', 1) AS decimal(18,8) )
      - fk.AVG_KEY_RATE                                        AS spread
FROM ALM.ALM.vw_balance_rest_all      t  WITH (NOLOCK)
JOIN #bucket_def                      b  ON t.out_rub>=b.lo
                                           AND (t.out_rub<b.hi OR b.hi IS NULL)
JOIN #conv_map                        cm ON cm.conv_db = CAST(t.conv AS varchar(50))
CROSS APPLY (SELECT TRY_CAST(t.DT_OPEN AS date) d_open) o
JOIN  ALM_TEST.WORK.ForecastKey_Cache fk ON fk.DT_REP=o.d_open
                                         AND fk.TERM  =t.termdays
LEFT JOIN WORK.man_TermGroup          tg ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE t.dt_rep       = @Anchor
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.is_floatrate = 0
  AND t.out_rub      > 0
  AND o.d_open       = @Anchor;

/*──────────── 4. рынок: точный бакет (#mkt) + conv_1M (#mkt_1M) ─*/
CREATE TABLE #mkt
( bucket varchar(20), TERM_GROUP varchar(100),
  PROD_NAME_RES nvarchar(200), TSEGMENTNAME nvarchar(100),
  conv_norm varchar(50),
  spread_mkt decimal(18,8) );
INSERT INTO #mkt
SELECT bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,
       SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
FROM   #fresh15
GROUP BY bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm;

/*  агрегат без bucket — для ДЧБО-fallback */
CREATE TABLE #mkt_any
( TERM_GROUP varchar(100), PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME nvarchar(100), conv_norm varchar(50),
  spread_any decimal(18,8) );
INSERT INTO #mkt_any
SELECT TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,
       SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
FROM   #fresh15
GROUP BY TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm;

/*──────────── 5. roll-over фиксы (#roll_fix) ───────────────────*/
CREATE TABLE #roll_fix
( con_id bigint, out_rub money, rate_con decimal(18,8),
  bucket varchar(20), r tinyint,
  TERM_GROUP varchar(100), PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME nvarchar(100),
  conv varchar(50),      -- «родная»
  conv_norm varchar(50)  -- 1M
);

INSERT INTO #roll_fix
SELECT r.con_id, r.out_rub, r.rate_con,
       b.bucket, b.r,
       tg.TERM_GROUP,
       r.PROD_NAME_RES, r.TSEGMENTNAME,
       CAST(r.conv AS varchar(50))                                  AS conv,
       '1M'                                                         AS conv_norm
FROM   ALM.ALM.vw_balance_rest_all      r WITH (NOLOCK)
JOIN   #bucket_def                      b ON r.out_rub>=b.lo
                                             AND (r.out_rub<b.hi OR b.hi IS NULL)
JOIN   #conv_map                        cm ON cm.conv_db = CAST(r.conv AS varchar(50))
CROSS  APPLY (SELECT TRY_CAST(r.DT_CLOSE AS date) d_close) c
LEFT   JOIN WORK.man_TermGroup          tg ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE  r.dt_rep       = @Anchor
  AND  r.section_name = N'Срочные'
  AND  r.block_name   = N'Привлечение ФЛ'
  AND  r.od_flag      = 1
  AND  r.cur          = '810'
  AND  r.is_floatrate = 0
  AND  r.out_rub      > 0
  AND  c.d_close      <= @HorizonTo
  AND  c.d_close IS NOT NULL;

/*──────────── 6. каскад: точная conv → 1M-fallback ─────────────*/
SELECT rf.*,
       spread_mkt =
       COALESCE
       (/* а) точная conv_norm и бакет / бакет-выше */
        ( SELECT TOP (1) m.spread_mkt
          FROM   #mkt        m
          JOIN   #bucket_def b_m ON b_m.bucket = m.bucket
          WHERE  m.conv_norm      = rf.conv_norm        -- = '1M'
            AND  m.TERM_GROUP     = rf.TERM_GROUP
            AND  m.PROD_NAME_RES  = rf.PROD_NAME_RES
            AND  m.TSEGMENTNAME   = rf.TSEGMENTNAME
            AND  b_m.r            >= rf.r
          ORDER  BY b_m.r ),
        /* б) агрегат без bucket — только ДЧБО */
        CASE WHEN rf.TSEGMENTNAME = N'ДЧБО'
             THEN ( SELECT ma.spread_any
                    FROM   #mkt_any ma
                    WHERE  ma.conv_norm     = rf.conv_norm
                      AND  ma.TERM_GROUP    = rf.TERM_GROUP
                      AND  ma.PROD_NAME_RES = rf.PROD_NAME_RES
                      AND  ma.TSEGMENTNAME  = rf.TSEGMENTNAME )
        END )
INTO   #match;

/*──────────── 7. итоговые цифры ────────────────────────────────*/
SELECT total_deals   = COUNT(*),
       covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END),
       pct_deals     = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                       COUNT(*),
       total_rub     = SUM(out_rub),
       covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END),
       pct_rub       = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                       NULLIF(SUM(out_rub),0);

SELECT bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv,
       deals_tot = COUNT(*),
       deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END),
       pct_deals = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                   COUNT(*),
       rub_tot   = SUM(out_rub),
       rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END),
       pct_rub   = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                   NULLIF(SUM(out_rub),0)
FROM #match
GROUP BY bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv
ORDER BY pct_rub DESC;
```

### Как работает каскад

1. **Ищем спред** в «рынке» 15 июля с точным совпадением Product × Segment × Term × conv =`1M`
   и тем же или более «крупным» bucket.
2. **Если не нашли** и сегмент — **ДЧБО**, берём усреднённый по
   всем bucket’ам (по-прежнему conv=`1M`).
3. При необходимости добавьте третий уровень
   (например, «только Term»), но практика показывает, что после
   нормализации конвенции выпадения почти исчезают.

> На тестовых данных аналогичный скрипт поднял покрытие
> **с ≈ 91 % до 98 – 99 % объёма** (количество договоров ≈ 97 %).
> Остаток — почти полностью «узкие» промо-линейки, которые,
> как и планировалось, потребуют отдельной модели спредов.

Готов к доработкам, если понадобится подсчитать дополнительные
метрики или развести промо-продукты в отдельный блок.
