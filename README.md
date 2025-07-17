/*──────── 4. roll-over фиксы  ────────*/
IF OBJECT_ID('tempdb..#roll_fix') IS NOT NULL DROP TABLE #roll_fix;

CREATE TABLE #roll_fix          -- ЯВНО объявляем структуру
( con_id        bigint,
  out_rub       money,
  bucket        varchar(20),
  r             tinyint,
  TERM_GROUP    varchar(100),
  PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME  nvarchar(100),
  conv_norm     varchar(50),
  spread_fix    decimal(18,6)   -- фактический спред сделки
);

INSERT INTO #roll_fix
( con_id,out_rub,bucket,r,
  TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,spread_fix)
SELECT  r.con_id,
        r.out_rub,
        b.bucket,
        b.r,
        tg.TERM_GROUP,
        r.PROD_NAME_RES,
        r.TSEGMENTNAME,
        CASE WHEN r.conv = 'AT_THE_END' THEN '1M'
             ELSE CAST(r.conv AS varchar(50)) END,
        r.rate_con - fk_open.AVG_KEY_RATE          -- фактический спред
FROM    ALM.ALM.vw_balance_rest_all      r  WITH (NOLOCK)
JOIN    #bucket_def                      b
          ON r.out_rub BETWEEN b.lo AND ISNULL(b.hi, r.out_rub)
LEFT JOIN WORK.man_TermGroup             tg
          ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fk_open
          ON fk_open.DT_REP = CAST(r.DT_OPEN AS date)
         AND fk_open.TERM   = r.termdays
CROSS APPLY (SELECT TRY_CAST(r.DT_CLOSE AS date) d_close) c
WHERE   r.dt_rep       = @Anchor
  AND   r.section_name = N'Срочные'
  AND   r.block_name   = N'Привлечение ФЛ'
  AND   r.od_flag      = 1
  AND   r.cur          = '810'
  AND   r.is_floatrate = 0
  AND   r.out_rub      > 0
  AND   c.d_close      <= @HorizonTo
  AND   c.d_close IS NOT NULL;

/*──────── 5. каскадное сопоставление ────────*/
IF OBJECT_ID('tempdb..#match') IS NOT NULL DROP TABLE #match;

CREATE TABLE #match            -- схема заранее
( con_id        bigint,
  out_rub       money,
  bucket        varchar(20),
  r             tinyint,
  TERM_GROUP    varchar(100),
  PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME  nvarchar(100),
  conv_norm     varchar(50),
  spread_fix    decimal(18,6),    -- фактический
  spread_mkt    decimal(18,6),    -- найденный «рыночный»
  spread_final  decimal(18,6),    -- что пойдёт в расчёт
  is_matched    bit               -- 1 = нашли сопоставление
);

INSERT INTO #match
( con_id,out_rub,bucket,r,
  TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,
  spread_fix,spread_mkt,spread_final,is_matched )
SELECT
       rf.con_id, rf.out_rub, rf.bucket, rf.r,
       rf.TERM_GROUP, rf.PROD_NAME_RES, rf.TSEGMENTNAME, rf.conv_norm,
       rf.spread_fix,
       /* 1. «рыночный» спред (точный бакет / ↑-бакет / fallback ДЧБО) */
       sp_mkt.spread_mkt,
       /* 2. финальный спред */
       COALESCE(sp_mkt.spread_mkt, rf.spread_fix)            AS spread_final,
       /* 3. флаг — нашли ли замену */
       CASE WHEN sp_mkt.spread_mkt IS NULL THEN 0 ELSE 1 END AS is_matched
FROM  #roll_fix rf
OUTER APPLY
(
    /* одно вычисление — потом используем дважды */
    SELECT TOP 1
           COALESCE(m1.spread_mkt, m2.spread_any) AS spread_mkt
    FROM   -- (а) тот же или более крупный бакет
           ( SELECT m.spread_mkt, b_m.r
             FROM   #mkt m
             JOIN   #bucket_def b_m ON b_m.bucket = m.bucket
             WHERE  m.TERM_GROUP     = rf.TERM_GROUP
               AND  m.PROD_NAME_RES  = rf.PROD_NAME_RES
               AND  m.TSEGMENTNAME   = rf.TSEGMENTNAME
               AND  m.conv_norm      = rf.conv_norm
               AND  b_m.r            >= rf.r )             m1
           FULL OUTER JOIN
           -- (б) fallback для ДЧБО
           ( SELECT ma.spread_any
             FROM   #mkt_any ma
             WHERE  rf.TSEGMENTNAME = N'ДЧБО'
               AND  ma.TERM_GROUP    = rf.TERM_GROUP
               AND  ma.PROD_NAME_RES = rf.PROD_NAME_RES
               AND  ma.TSEGMENTNAME  = rf.TSEGMENTNAME
               AND  ma.conv_norm     = rf.conv_norm )      m2
             ON 1 = 1
    ORDER BY r                                    -- ближайший крупный бакет
) sp_mkt;
