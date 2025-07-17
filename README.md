Ниже – полностью переписанный скрипт (одна вставка).
Все имена столбцов взяты **строго** из `vw_balance_rest_all` и справочников;
в спорных местах я обрамил их `[]`, чтобы парсер не путался.

```sql
/*─────────────────────────  ПАРАМЕТРЫ  ─────────────────────────*/
DECLARE
    @Anchor     date = '2025-07-15',          -- фактический dt_rep
    @HorizonTo  date = '2025-08-31';          -- край для roll-over

/*────────── housekeeping ──────────*/
IF OBJECT_ID('tempdb..#fresh15')     IS NOT NULL DROP TABLE #fresh15;
IF OBJECT_ID('tempdb..#mkt')         IS NOT NULL DROP TABLE #mkt;
IF OBJECT_ID('tempdb..#mkt_any')     IS NOT NULL DROP TABLE #mkt_any;
IF OBJECT_ID('tempdb..#bucket_rank') IS NOT NULL DROP TABLE #bucket_rank;
IF OBJECT_ID('tempdb..#roll_fix')    IS NOT NULL DROP TABLE #roll_fix;
IF OBJECT_ID('tempdb..#match')       IS NOT NULL DROP TABLE #match;

/*────────── 0. шкала бакетов ──────────*/
CREATE TABLE #bucket_rank(bucket varchar(20) PRIMARY KEY, r int);
INSERT #bucket_rank VALUES
('[0-1.5 млн)',0),('[1.5-15 млн)',1),('[15-100 млн)',2),('[100 млн+]',3);

/*────────── 1. рынок = фиксы, открытые 15-го ──────────*/
SELECT
        bucket =
             CASE WHEN t.[out_rub] <  1500000     THEN '[0-1.5 млн)'
                  WHEN t.[out_rub] < 15000000     THEN '[1.5-15 млн)'
                  WHEN t.[out_rub] <100000000     THEN '[15-100 млн)'
                  ELSE                               '[100 млн+]' END,
        tg.[TERM_GROUP],
        t.[PROD_NAME_RES],
        t.[TSEGMENTNAME],
        [conv] = CAST(t.[conv] AS varchar(50)),
        t.[out_rub],
        spread = t.[rate_con] - fk.[AVG_KEY_RATE]
INTO    #fresh15
FROM    ALM.ALM.vw_balance_rest_all      AS t  WITH (NOLOCK)
CROSS  APPLY (SELECT TRY_CAST(t.[DT_OPEN] AS date) AS d_open) AS o
JOIN    ALM_TEST.WORK.ForecastKey_Cache  AS fk
           ON fk.[DT_REP] = o.d_open
          AND fk.[TERM]   = t.[termdays]
LEFT JOIN WORK.man_TermGroup            AS tg
           ON t.[termdays] BETWEEN tg.[TERM_FROM] AND tg.[TERM_TO]
WHERE   t.[dt_rep]      = @Anchor
  AND   t.[section_name]= N'Срочные'
  AND   t.[block_name]  = N'Привлечение ФЛ'
  AND   t.[od_flag]     = 1
  AND   t.[cur]         = '810'
  AND   t.[is_floatrate]= 0
  AND   t.[out_rub]     > 0
  AND   o.d_open        = @Anchor
  AND   o.d_open IS NOT NULL;

/*────────── 2. рынок-спред (точный, по бакету) ──────────*/
SELECT   bucket, [TERM_GROUP], [PROD_NAME_RES], [TSEGMENTNAME], [conv],
         spread_mkt = SUM([out_rub]*spread)/NULLIF(SUM([out_rub]),0)
INTO     #mkt
FROM     #fresh15
GROUP BY bucket, [TERM_GROUP], [PROD_NAME_RES], [TSEGMENTNAME], [conv];

/*────────── 3. рынок-спред без-бакетный (для ДЧБО-fallback) ─────*/
SELECT   [TERM_GROUP], [PROD_NAME_RES], [TSEGMENTNAME], [conv],
         spread_any = SUM([out_rub]*spread)/NULLIF(SUM([out_rub]),0)
INTO     #mkt_any
FROM     #fresh15
GROUP BY [TERM_GROUP], [PROD_NAME_RES], [TSEGMENTNAME], [conv];

/*────────── 4. roll-over фиксы ≤ 31-08 ──────────*/
SELECT
        rf.[con_id],
        rf.[out_rub],
        rf.[rate_con],
        bucket =
             CASE WHEN rf.[out_rub] <  1500000     THEN '[0-1.5 млн)'
                  WHEN rf.[out_rub] < 15000000     THEN '[1.5-15 млн)'
                  WHEN rf.[out_rub] <100000000     THEN '[15-100 млн)'
                  ELSE                                '[100 млн+]' END,
        tg.[TERM_GROUP],
        rf.[PROD_NAME_RES],
        rf.[TSEGMENTNAME],
        [conv] = CAST(rf.[conv] AS varchar(50))
INTO    #roll_fix
FROM    ALM.ALM.vw_balance_rest_all      AS rf WITH (NOLOCK)
CROSS  APPLY (SELECT TRY_CAST(rf.[DT_CLOSE] AS date) AS d_close) AS c
LEFT JOIN WORK.man_TermGroup             AS tg
           ON rf.[termdays] BETWEEN tg.[TERM_FROM] AND tg.[TERM_TO]
WHERE   rf.[dt_rep]      = @Anchor
  AND   rf.[section_name]= N'Срочные'
  AND   rf.[block_name]  = N'Привлечение ФЛ'
  AND   rf.[od_flag]     = 1
  AND   rf.[cur]         = '810'
  AND   rf.[is_floatrate]= 0
  AND   rf.[out_rub]     > 0
  AND   c.d_close        <= @HorizonTo
  AND   c.d_close IS NOT NULL;

/*────────── 5. каскадный поиск спреда ──────────*/
SELECT  r.*,
        spread_mkt = COALESCE
        ( /* 5-a. точный или ↑-бакетный поиск */
          ( SELECT TOP 1 m.spread_mkt
            FROM   #mkt               AS m
            JOIN   #bucket_rank       AS b_m ON b_m.bucket = m.bucket
            JOIN   #bucket_rank       AS b_r ON b_r.bucket = r.bucket
            WHERE  m.[TERM_GROUP]     = r.[TERM_GROUP]
              AND  m.[PROD_NAME_RES]  = r.[PROD_NAME_RES]
              AND  m.[TSEGMENTNAME]   = r.[TSEGMENTNAME]
              AND  m.[conv]           = r.[conv]
              AND  b_m.r              >= b_r.r     -- тот же или ВЫШЕ
            ORDER BY b_m.r            -- ближайший больший
          ),
          /* 5-b. fallback только для ДЧБО */
          CASE WHEN r.[TSEGMENTNAME] = N'ДЧБО'
               THEN (SELECT ma.spread_any
                     FROM   #mkt_any AS ma
                     WHERE  ma.[TERM_GROUP]    = r.[TERM_GROUP]
                       AND  ma.[PROD_NAME_RES] = r.[PROD_NAME_RES]
                       AND  ma.[TSEGMENTNAME]  = r.[TSEGMENTNAME]
                       AND  ma.[conv]          = r.[conv])
          END )
INTO    #match
FROM    #roll_fix AS r;

/*────────── 6. общая сводка ──────────*/
SELECT
    total_deals   = COUNT(*) ,
    covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals     = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)/COUNT(*) ,
    total_rub     = SUM([out_rub]) ,
    covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN [out_rub] ELSE 0 END) ,
    pct_rub       = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN [out_rub] ELSE 0 END)
/                   NULLIF(SUM([out_rub]),0)
FROM #match;

/*────────── 7. детализация (5 мер + conv) ──────────*/
SELECT
    bucket,
    [TERM_GROUP],
    [PROD_NAME_RES],
    [TSEGMENTNAME],
    [conv],
    deals_tot = COUNT(*) ,
    deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)/COUNT(*) ,
    rub_tot   = SUM([out_rub]) ,
    rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN [out_rub] ELSE 0 END) ,
    pct_rub   = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN [out_rub] ELSE 0 END)
/                NULLIF(SUM([out_rub]),0)
FROM  #match
GROUP BY bucket, [TERM_GROUP], [PROD_NAME_RES], [TSEGMENTNAME], [conv]
ORDER BY pct_rub DESC;
```

### Что изменилось

| Блок                 | Коррекция                                                                                             |
| -------------------- | ----------------------------------------------------------------------------------------------------- |
| <br>**Все SELECT-ы** | имена столбцов взяты точно из витрины и обрамлены `[]`, чтобы парсер не считал их зарезервированными. |
| 1, 4                 | `CROSS APPLY TRY_CAST(... AS date)` — безопасно конвертируем строковое `DT_OPEN/DT_CLOSE`.            |
| 5                    | каскад (`COALESCE`) остаётся тем же, но опирается на выровненные имена.                               |

Скопируйте / вставьте целиком — парсер больше не должен жаловаться на «недопустимые имена столбцов».
