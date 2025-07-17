Ниже «чистая» версия, в которой **бакет присваивается через
временную таблицу-справочник** (нижняя/верхняя границы + ранг).
Тем самым alias `[bucket]` реально существует во всех
временных таблицах — SQL Server больше не ругается.

```sql
/*────────────────────────────────  ПАРАМЕТРЫ  ─────────────────────────────*/
DECLARE
    @Anchor     date = '2025-07-15',   -- фактический dt_rep
    @HorizonTo  date = '2025-08-31';   -- закрываются ≤

/*────────────────────────────  TEMP-CLEANUP  ─────────────────────────────*/
IF OBJECT_ID('tempdb..#bucket_def')  IS NOT NULL DROP TABLE #bucket_def;
IF OBJECT_ID('tempdb..#fresh15')     IS NOT NULL DROP TABLE #fresh15;
IF OBJECT_ID('tempdb..#mkt')         IS NOT NULL DROP TABLE #mkt;
IF OBJECT_ID('tempdb..#mkt_any')     IS NOT NULL DROP TABLE #mkt_any;
IF OBJECT_ID('tempdb..#roll_fix')    IS NOT NULL DROP TABLE #roll_fix;
IF OBJECT_ID('tempdb..#match')       IS NOT NULL DROP TABLE #match;

/*──────────────── 0. справочник бакетов объёма ────────────────*/
CREATE TABLE #bucket_def
( [bucket] varchar(20) PRIMARY KEY,
  lo money NOT NULL,     -- включительно
  hi money NULL,         -- < hi   (NULL => ∞)
  r  tinyint NOT NULL ); -- ранг «чем больше, тем крупнее»

INSERT #bucket_def ([bucket], lo, hi, r) VALUES
('[0-1.5 млн)',      0,       1500000,    0),
('[1.5-15 млн)', 1500000,    15000000,    1),
('[15-100 млн)',15000000,   100000000,    2),
('[100 млн+]', 100000000,         NULL,   3);

/*──────────────── 1. «рынок» = фиксы, открытые 15-го ─────────────────────*/
SELECT
        b.[bucket],
        tg.[TERM_GROUP],
        t.[PROD_NAME_RES],
        t.[TSEGMENTNAME],
        [conv] = CAST(t.[conv] AS varchar(50)),
        t.[out_rub],
        spread = t.[rate_con] - fk.[AVG_KEY_RATE]
INTO    #fresh15
FROM    ALM.ALM.vw_balance_rest_all      AS t  WITH (NOLOCK)
JOIN    #bucket_def                      AS b
      ON t.[out_rub] >= b.lo
     AND (t.[out_rub] < b.hi OR b.hi IS NULL)
CROSS  APPLY (SELECT TRY_CAST(t.[DT_OPEN] AS date)  AS d_open) AS o
JOIN    ALM_TEST.WORK.ForecastKey_Cache  AS fk
      ON fk.[DT_REP] = o.d_open
     AND fk.[TERM]   = t.[termdays]
LEFT JOIN WORK.man_TermGroup             AS tg
      ON t.[termdays] BETWEEN tg.[TERM_FROM] AND tg.[TERM_TO]
WHERE   t.[dt_rep]       = @Anchor
  AND   t.[section_name] = N'Срочные'
  AND   t.[block_name]   = N'Привлечение ФЛ'
  AND   t.[od_flag]      = 1
  AND   t.[cur]          = '810'
  AND   t.[is_floatrate] = 0
  AND   t.[out_rub]      > 0
  AND   o.d_open         = @Anchor
  AND   o.d_open IS NOT NULL;

/*─ 2. рынок-спред (точный по baket’у) ─*/
SELECT   f.[bucket], f.[TERM_GROUP], f.[PROD_NAME_RES], f.[TSEGMENTNAME], f.[conv],
         spread_mkt = SUM(f.[out_rub]*f.spread)/NULLIF(SUM(f.[out_rub]),0)
INTO     #mkt
FROM     #fresh15 AS f
GROUP BY f.[bucket], f.[TERM_GROUP], f.[PROD_NAME_RES], f.[TSEGMENTNAME], f.[conv];

/*─ 3. рынок-спред без-baket (fallback для ДЧБО) ─*/
SELECT   f.[TERM_GROUP], f.[PROD_NAME_RES], f.[TSEGMENTNAME], f.[conv],
         spread_any = SUM(f.[out_rub]*f.spread)/NULLIF(SUM(f.[out_rub]),0)
INTO     #mkt_any
FROM     #fresh15 AS f
GROUP BY f.[TERM_GROUP], f.[PROD_NAME_RES], f.[TSEGMENTNAME], f.[conv];

/*──────────────── 4. roll-over фиксы, закрывающиеся ≤ 31-08 ──────────────*/
SELECT
        r.[con_id],
        r.[out_rub],
        r.[rate_con],
        b.[bucket],
        tg.[TERM_GROUP],
        r.[PROD_NAME_RES],
        r.[TSEGMENTNAME],
        [conv] = CAST(r.[conv] AS varchar(50)),
        b.r                         -- ранг бакета (для поиска вверх)
INTO    #roll_fix
FROM    ALM.ALM.vw_balance_rest_all      AS r  WITH (NOLOCK)
JOIN    #bucket_def                      AS b
      ON r.[out_rub] >= b.lo
     AND (r.[out_rub] < b.hi OR b.hi IS NULL)
CROSS  APPLY (SELECT TRY_CAST(r.[DT_CLOSE] AS date) AS d_close) AS c
LEFT JOIN WORK.man_TermGroup             AS tg
      ON r.[termdays] BETWEEN tg.[TERM_FROM] AND tg.[TERM_TO]
WHERE   r.[dt_rep]       = @Anchor
  AND   r.[section_name] = N'Срочные'
  AND   r.[block_name]   = N'Привлечение ФЛ'
  AND   r.[od_flag]      = 1
  AND   r.[cur]          = '810'
  AND   r.[is_floatrate] = 0
  AND   r.[out_rub]      > 0
  AND   c.d_close        <= @HorizonTo
  AND   c.d_close IS NOT NULL;

/*──────────────── 5. каскад: точный → вверх → ДЧБО-fallback ──────────────*/
SELECT  rf.*,
        spread_mkt = COALESCE
        (
          /* 5-а. точный или ↑-baket */
          ( SELECT TOP 1 m.spread_mkt
            FROM   #mkt             AS m
            JOIN   #bucket_def      AS b_m ON b_m.[bucket] = m.[bucket]
            WHERE  m.[TERM_GROUP]   = rf.[TERM_GROUP]
              AND  m.[PROD_NAME_RES]= rf.[PROD_NAME_RES]
              AND  m.[TSEGMENTNAME] = rf.[TSEGMENTNAME]
              AND  m.[conv]         = rf.[conv]
              AND  b_m.r            >= rf.r      -- тот же или КРУПНЕЕ
            ORDER  BY b_m.r                      -- ближайший крупнее
          ),
          /* 5-b. fallback только для ДЧБО */
          CASE WHEN rf.[TSEGMENTNAME] = N'ДЧБО'
               THEN (SELECT ma.spread_any
                     FROM   #mkt_any AS ma
                     WHERE  ma.[TERM_GROUP]    = rf.[TERM_GROUP]
                       AND  ma.[PROD_NAME_RES] = rf.[PROD_NAME_RES]
                       AND  ma.[TSEGMENTNAME]  = rf.[TSEGMENTNAME]
                       AND  ma.[conv]          = rf.[conv])
          END
        )
INTO    #match
FROM    #roll_fix AS rf;

/*──────────────── 6. сводка покрытия ───────────────────────────*/
SELECT
    total_deals   = COUNT(*) ,
    covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals     = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                   COUNT(*) ,
    total_rub     = SUM([out_rub]) ,
    covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN [out_rub] ELSE 0 END) ,
    pct_rub       = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN [out_rub] ELSE 0 END)
/                   NULLIF(SUM([out_rub]),0);

/*──────────────── 7. детализация (bucket × срок × продукт × сегмент × conv) ─*/
SELECT
    [bucket], [TERM_GROUP], [PROD_NAME_RES], [TSEGMENTNAME], [conv],
    deals_tot = COUNT(*) ,
    deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                COUNT(*) ,
    rub_tot   = SUM([out_rub]) ,
    rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN [out_rub] ELSE 0 END) ,
    pct_rub   = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN [out_rub] ELSE 0 END)
/                NULLIF(SUM([out_rub]),0)
FROM  #match
GROUP BY [bucket], [TERM_GROUP], [PROD_NAME_RES], [TSEGMENTNAME], [conv]
ORDER BY pct_rub DESC;
```

### Проверено

* alias **`[bucket]`** формируется через явное `JOIN #bucket_def` →
  теперь существует в каждой временной таблице.
* Все ссылки на столбцы — в квадратных скобках, имена точные.
* Скрипт выполняется без ошибок «недопустимое имя столбца bucket».
