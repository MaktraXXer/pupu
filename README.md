Ниже — целый скрипт, уже с явным перечислением столбцов в каждом `INSERT … (<col-list>)` — тогда ошибка
«имя столбца или число представленных значений …» не возникает даже при строгой проверке.

```sql
/*────────────────────────  0. параметры  ───────────────────────*/
DECLARE
    @Anchor     date = '2025-07-15',
    @HorizonTo  date = '2025-08-31';

/*──────────────────  0.1 очистка TEMP  ─────────────────────────*/
IF OBJECT_ID('tempdb..#bucket_def') IS NOT NULL DROP TABLE #bucket_def;
IF OBJECT_ID('tempdb..#fresh15')    IS NOT NULL DROP TABLE #fresh15;
IF OBJECT_ID('tempdb..#mkt')        IS NOT NULL DROP TABLE #mkt;
IF OBJECT_ID('tempdb..#mkt_any')    IS NOT NULL DROP TABLE #mkt_any;
IF OBJECT_ID('tempdb..#roll_fix')   IS NOT NULL DROP TABLE #roll_fix;
IF OBJECT_ID('tempdb..#match')      IS NOT NULL DROP TABLE #match;

SET NOCOUNT ON;

/*────────────────────  1. бакеты объёма  ──────────────────────*/
CREATE TABLE #bucket_def
( bucket varchar(20) PRIMARY KEY,
  lo     money NOT NULL,
  hi     money NULL,
  r      tinyint NOT NULL);

INSERT #bucket_def (bucket,lo,hi,r) VALUES
('[0-1.5 млн)',0,1500000,0),
('[1.5-15 млн)',1500000,15000000,1),
('[15-100 млн)',15000000,100000000,2),
('[100 млн+]',100000000,NULL,3);

/*────────────────────  2. fresh-выдачи 15-го  ──────────────────*/
CREATE TABLE #fresh15
( bucket        varchar(20),
  TERM_GROUP    varchar(100),
  PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME  nvarchar(100),
  conv_norm     varchar(50),
  out_rub       money,
  spread        decimal(18,6));

/* 2-а. «живые» месячные / прочие конвенции */
INSERT #fresh15 (bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,out_rub,spread)
SELECT
        b.bucket,
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,
        CAST(t.conv AS varchar(50))          AS conv_norm,
        t.out_rub,
        t.rate_con - fk.AVG_KEY_RATE         AS spread
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
JOIN    #bucket_def b ON t.out_rub>=b.lo AND (t.out_rub<b.hi OR b.hi IS NULL)
CROSS APPLY (SELECT TRY_CAST(t.DT_OPEN AS date) d_open) o
JOIN    ALM_TEST.WORK.ForecastKey_Cache fk
              ON fk.DT_REP=o.d_open AND fk.TERM=t.termdays
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
  AND   t.conv        <> 'AT_THE_END';

/* 2-б. «виртуальная» 1M для ставок AT_THE_END */
INSERT #fresh15 (bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,out_rub,spread)
SELECT
        b.bucket,
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,
        '1M'                                   AS conv_norm,
        t.out_rub,
        CAST([LIQUIDITY].[liq].[fnc_IntRate]
                ( t.rate_con, 'at the end','monthly',t.termdays,1 )
             AS decimal(18,6)) - fk.AVG_KEY_RATE   AS spread
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
JOIN    #bucket_def b ON t.out_rub>=b.lo AND (t.out_rub<b.hi OR b.hi IS NULL)
CROSS APPLY (SELECT TRY_CAST(t.DT_OPEN AS date) d_open) o
JOIN    ALM_TEST.WORK.ForecastKey_Cache fk
             ON fk.DT_REP=o.d_open AND fk.TERM=t.termdays
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
  AND   t.conv         = 'AT_THE_END';

/*────────────────────  3. справочники спредов  ─────────────────*/
SELECT
    bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm,
    spread_mkt = SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
INTO #mkt
FROM #fresh15
GROUP BY bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm;

SELECT
    TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm,
    spread_any = SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
INTO #mkt_any
FROM #fresh15
GROUP BY TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm;

/*────────────────────  4. roll-over-фиксы  ─────────────────────*/
CREATE TABLE #roll_fix
( con_id        bigint,
  out_rub       money,
  rate_con      decimal(18,6),
  bucket        varchar(20),
  r             tinyint,
  TERM_GROUP    varchar(100),
  PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME  nvarchar(100),
  conv_norm     varchar(50));

INSERT #roll_fix
        (con_id,out_rub,rate_con,bucket,r,TERM_GROUP,
         PROD_NAME_RES,TSEGMENTNAME,conv_norm)
SELECT r.con_id,r.out_rub,r.rate_con,
       b.bucket,b.r,
       tg.TERM_GROUP,
       r.PROD_NAME_RES,r.TSEGMENTNAME,
       CASE WHEN r.conv='AT_THE_END' THEN '1M'
            ELSE CAST(r.conv AS varchar(50)) END  AS conv_norm
FROM   ALM.ALM.vw_balance_rest_all r WITH (NOLOCK)
JOIN   #bucket_def b ON r.out_rub>=b.lo AND (r.out_rub<b.hi OR b.hi IS NULL)
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
  AND  c.d_close      IS NOT NULL;

/*────────────────────  5. сопоставление  ───────────────────────*/
SELECT rf.*,
       spread_mkt =
       COALESCE
       ( /* a) точный / «крупнее» бакет */
         (SELECT TOP (1) m.spread_mkt
            FROM #mkt AS m
            JOIN #bucket_def b_m ON b_m.bucket=m.bucket
           WHERE m.TERM_GROUP     = rf.TERM_GROUP
             AND m.PROD_NAME_RES  = rf.PROD_NAME_RES
             AND m.TSEGMENTNAME   = rf.TSEGMENTNAME
             AND m.conv_norm      = rf.conv_norm
             AND b_m.r            >= rf.r
           ORDER BY b_m.r),
         /* b) fallback среднее (ДЧБО) */
         CASE WHEN rf.TSEGMENTNAME=N'ДЧБО'
              THEN (SELECT ma.spread_any
                      FROM #mkt_any ma
                     WHERE ma.TERM_GROUP    = rf.TERM_GROUP
                       AND ma.PROD_NAME_RES = rf.PROD_NAME_RES
                       AND ma.TSEGMENTNAME  = rf.TSEGMENTNAME
                       AND ma.conv_norm     = rf.conv_norm)
         END)
INTO #match
FROM #roll_fix rf;

/*────────────────────  6. сводка  ──────────────────────────────*/
SELECT
    total_deals   = COUNT(*),
    covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END),
    pct_deals     = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                   COUNT(*),
    total_rub     = SUM(out_rub),
    covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END),
    pct_rub       = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                   NULLIF(SUM(out_rub),0);

/*────────────────────  7. детализация  ─────────────────────────*/
SELECT
    bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm,
    deals_tot = COUNT(*),
    deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END),
    pct_deals = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                COUNT(*),
    rub_tot   = SUM(out_rub),
    rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END),
    pct_rub   = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                NULLIF(SUM(out_rub),0)
FROM #match
GROUP BY bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm
ORDER BY pct_rub DESC;
```

### что изменилось

| Блок         | Изменение                                                          | Зачем                                                                        |
| ------------ | ------------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| **INSERT-ы** | везде указан явный список столбцов -- `(bucket, … , spread)`       | SQL Server больше не проверяет «по порядку» — совпадает строго по количеству |
| 2-а, 2-б     | `conv_norm` всегда задан альясом (или `'1M'`)                      | таблица принимает точно 7 колонок                                            |
| остальное    | без изменений к логике (91 % прежних + «нормализация» для остатка) | покрытие должно вырасти, ошибки имён исчезают                                |

Запустите скрипт одной вставкой — сводка (шаг 6) и детализация (шаг 7) появятся сразу.
