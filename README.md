/*────────────────────────  0. ПАРАМЕТРЫ  ───────────────────────*/
DECLARE
    @Anchor     date = '2025-07-15',      -- последний факт
    @HorizonTo  date = '2025-08-31';      -- закрываются ≤

/*────────────────────  0.1 TEMP-CLEANUP  ──────────────────────*/
IF OBJECT_ID('tempdb..#bucket_def') IS NOT NULL DROP TABLE #bucket_def;
IF OBJECT_ID('tempdb..#fresh15')    IS NOT NULL DROP TABLE #fresh15;
IF OBJECT_ID('tempdb..#mkt')        IS NOT NULL DROP TABLE #mkt;
IF OBJECT_ID('tempdb..#mkt_any')    IS NOT NULL DROP TABLE #mkt_any;
IF OBJECT_ID('tempdb..#roll_fix')   IS NOT NULL DROP TABLE #roll_fix;
IF OBJECT_ID('tempdb..#match')      IS NOT NULL DROP TABLE #match;

/*────────────────────  1. Справочник бакетов  ─────────────────*/
CREATE TABLE #bucket_def
( bucket varchar(20) PRIMARY KEY ,
  lo     money        NOT NULL ,  -- начало  (вкл.)
  hi     money        NULL ,      -- < hi ; NULL = ∞
  r      tinyint      NOT NULL ); -- ранг: чем больше, тем крупнее
INSERT #bucket_def VALUES
('[0-1.5 млн)',     0        ,1500000   ,0),
('[1.5-15 млн)',    1500000  ,15000000  ,1),
('[15-100 млн)',    15000000 ,100000000 ,2),
('[100 млн+]',      100000000,NULL      ,3);

/*──────────────  2. «Рынок» — фиксы, открытые 15-го  ──────────*/
CREATE TABLE #fresh15
( bucket         varchar(20),
  TERM_GROUP     varchar(100) ,
  PROD_NAME_RES  nvarchar(200),
  TSEGMENTNAME   nvarchar(100),
  conv_norm      varchar(50) ,       -- «AT_THE_END» → 1M
  out_rub        money,
  rate_adj       decimal(18,8),      -- ставка уже в 1M-конвенции
  spread         decimal(18,8) );    -- rate_adj − KEY

INSERT INTO #fresh15
SELECT
        b.bucket,
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,

        conv_norm =
            CASE WHEN t.conv = 'AT_THE_END' THEN '1M'
                 ELSE CAST(t.conv AS varchar(50)) END,

        t.out_rub,

        /* ставка в нужной конвенции */
        rate_adj = CAST(
                     CASE
                         WHEN t.conv = 'AT_THE_END'
                         THEN LIQUIDITY.liq.fnc_IntRate
                                ( t.rate_con,'at the end','monthly'
                                , t.termdays,1)
                         ELSE t.rate_con
                     END AS decimal(18,8)),

        /* спред к ключу (ключ всегда в 1M) */
        CAST(
          CASE
              WHEN t.conv = 'AT_THE_END'
              THEN LIQUIDITY.liq.fnc_IntRate
                     ( t.rate_con,'at the end','monthly'
                     , t.termdays,1)
              ELSE t.rate_con
          END AS decimal(18,8))
        - fk.AVG_KEY_RATE                               AS spread
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
JOIN    #bucket_def  b
      ON t.out_rub >= b.lo AND (t.out_rub < b.hi OR b.hi IS NULL)
CROSS  APPLY (SELECT TRY_CAST(t.DT_OPEN AS date) d_open) o
JOIN    ALM_TEST.WORK.ForecastKey_Cache fk
      ON fk.DT_REP = o.d_open AND fk.TERM = t.termdays
LEFT JOIN WORK.man_TermGroup tg
      ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE   t.dt_rep       = @Anchor
  AND   t.section_name = N'Срочные'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.is_floatrate = 0
  AND   t.out_rub      > 0
  AND   o.d_open       = @Anchor
  AND   o.d_open IS NOT NULL;

/*────────────  3. «Рынок»: точный & безбакетный  ──────────────*/
CREATE TABLE #mkt     /* точный: в своём бакете */
( bucket varchar(20), TERM_GROUP varchar(100),
  PROD_NAME_RES nvarchar(200), TSEGMENTNAME nvarchar(100),
  conv_norm varchar(50),
  spread_mkt decimal(18,8) );

INSERT INTO #mkt
SELECT bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,
       SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
FROM   #fresh15
GROUP  BY bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm;

CREATE TABLE #mkt_any         /* без бакета: fallback для ДЧБО */
( TERM_GROUP varchar(100), PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME nvarchar(100), conv_norm varchar(50),
  spread_any decimal(18,8) );

INSERT INTO #mkt_any
SELECT TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,
       SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
FROM   #fresh15
GROUP  BY TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm;

/*───────────────  4. Roll-over фиксы, закрывающиеся ≤ 31-08  ─────────────*/
CREATE TABLE #roll_fix
( con_id        bigint,
  out_rub       money,
  rate_adj      decimal(18,8),
  bucket        varchar(20),
  r             tinyint,       -- ранг бакета
  TERM_GROUP    varchar(100),
  PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME  nvarchar(100),
  conv_norm     varchar(50) );

INSERT INTO #roll_fix
SELECT r.con_id,
       r.out_rub,

       /* ставка в 1M-конвенции */
       rate_adj = CAST(
                   CASE
                       WHEN r.conv = 'AT_THE_END'
                       THEN LIQUIDITY.liq.fnc_IntRate
                              ( r.rate_con,'at the end','monthly'
                              , r.termdays,1)
                       ELSE r.rate_con
                   END AS decimal(18,8)),

       b.bucket, b.r,
       tg.TERM_GROUP,
       r.PROD_NAME_RES,
       r.TSEGMENTNAME,
       CASE WHEN r.conv = 'AT_THE_END' THEN '1M'
            ELSE CAST(r.conv AS varchar(50)) END          AS conv_norm
FROM   ALM.ALM.vw_balance_rest_all r  WITH (NOLOCK)
JOIN   #bucket_def b
     ON r.out_rub >= b.lo AND (r.out_rub < b.hi OR b.hi IS NULL)
CROSS APPLY (SELECT TRY_CAST(r.DT_CLOSE AS date) d_close) c
LEFT JOIN WORK.man_TermGroup tg
       ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE  r.dt_rep       = @Anchor
  AND  r.section_name = N'Срочные'
  AND  r.block_name   = N'Привлечение ФЛ'
  AND  r.od_flag      = 1
  AND  r.cur          = '810'
  AND  r.is_floatrate = 0
  AND  r.out_rub      > 0
  AND  c.d_close      <= @HorizonTo
  AND  c.d_close IS NOT NULL;

/*──────────────  5. Каскадное сопоставление  ──────────────*/
SELECT rf.*,
       spread_mkt = COALESCE
       ( /* а) тот же бакет или крупнее (conv уже нормализован) */
         (SELECT TOP (1) m.spread_mkt
          FROM   #mkt m
          JOIN   #bucket_def b_m ON b_m.bucket = m.bucket
          WHERE  m.TERM_GROUP      = rf.TERM_GROUP
            AND  m.PROD_NAME_RES   = rf.PROD_NAME_RES
            AND  m.TSEGMENTNAME    = rf.TSEGMENTNAME
            AND  m.conv_norm       = rf.conv_norm
            AND  b_m.r            >= rf.r          -- тот же или крупнее
          ORDER  BY b_m.r ),
         /* б) fallback «без бакета» – только для ДЧБО */
         CASE WHEN rf.TSEGMENTNAME = N'ДЧБО'
              THEN (SELECT ma.spread_any
                    FROM   #mkt_any ma
                    WHERE  ma.TERM_GROUP     = rf.TERM_GROUP
                      AND  ma.PROD_NAME_RES  = rf.PROD_NAME_RES
                      AND  ma.TSEGMENTNAME   = rf.TSEGMENTNAME
                      AND  ma.conv_norm      = rf.conv_norm)
         END )
INTO   #match
FROM   #roll_fix rf;

/*──────────────  6. Сводный отчёт  ──────────────*/
SELECT  total_deals   = COUNT(*) ,
        covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END),
        pct_deals     = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                       COUNT(*) ,
        total_rub     = SUM(out_rub) ,
        covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END),
        pct_rub       = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                       NULLIF(SUM(out_rub),0);
PRINT '── Детализация по bucket × срок × продукт × сегмент × conv_norm ──';
SELECT  bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm,
        deals_tot = COUNT(*) ,
        deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END),
        pct_deals = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                    COUNT(*) ,
        rub_tot   = SUM(out_rub) ,
        rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END),
        pct_rub   = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                    NULLIF(SUM(out_rub),0)
FROM    #match
GROUP  BY bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm
ORDER  BY pct_rub DESC;
