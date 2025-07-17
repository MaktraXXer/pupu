```sql
/*──────────────────────────  ПАРАМЕТРЫ  ──────────────────────────*/
DECLARE
    @Anchor     date = '2025-07-15',     -- фактический DT_REP
    @Prev       date = '2025-07-14',     -- день-резерв
    @HorizonTo  date = '2025-08-31';     -- крайний roll-over

/* ── Функция-выражение: новый бакет остатка ───────────────────────*/
-- 0–1.5 млн · 1.5–15 млн · 15–100 млн · 100 млн+
DECLARE @bucket_expr nvarchar(max) =
N'CASE
     WHEN t.out_rub < 1500000      THEN ''[0-1.5 млн)''
     WHEN t.out_rub < 15000000     THEN ''[1.5-15 млн)''
     WHEN t.out_rub < 100000000    THEN ''[15-100 млн)''
     ELSE                              ''[100 млн+]'' END';

/* ───────────────── Temp-таблицы cleanup ─────────────────────────*/
IF OBJECT_ID('tempdb..#fresh15')   IS NOT NULL DROP TABLE #fresh15;
IF OBJECT_ID('tempdb..#fresh14')   IS NOT NULL DROP TABLE #fresh14;
IF OBJECT_ID('tempdb..#mkt')       IS NOT NULL DROP TABLE #mkt;
IF OBJECT_ID('tempdb..#roll_fix')  IS NOT NULL DROP TABLE #roll_fix;
IF OBJECT_ID('tempdb..#match')     IS NOT NULL DROP TABLE #match;

/* ── 1. «свежие» фиксы 15-го июля ────────────────────────────────*/
SELECT
        BALANCE_GROUP = EXEC(@bucket_expr),
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,
        conv = CAST(t.conv AS varchar(50)),
        t.out_rub,
        spread = t.rate_con - fk.AVG_KEY_RATE
INTO    #fresh15
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
CROSS  APPLY (SELECT TRY_CAST(t.dt_open AS date) d_open) o
JOIN    ALM_TEST.WORK.ForecastKey_Cache fk
          ON fk.DT_REP = o.d_open
         AND fk.TERM   = t.termdays
LEFT JOIN WORK.man_TermGroup tg
          ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE   t.dt_rep = @Anchor
  AND   t.is_floatrate = 0
  AND   o.d_open = @Anchor
  AND   o.d_open IS NOT NULL;

/* ── 2. резерв 14-го июля (только непокрытые комбинации) ─────────*/
SELECT
        BALANCE_GROUP = EXEC(@bucket_expr),
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,
        conv = CAST(t.conv AS varchar(50)),
        t.out_rub,
        spread = t.rate_con - fk.AVG_KEY_RATE
INTO    #fresh14
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
CROSS  APPLY (SELECT TRY_CAST(t.dt_open AS date) d_open) o
JOIN    ALM_TEST.WORK.ForecastKey_Cache fk
          ON fk.DT_REP = o.d_open
         AND fk.TERM   = t.termdays
LEFT JOIN WORK.man_TermGroup tg
          ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
LEFT JOIN #fresh15 f
       ON  f.BALANCE_GROUP  = EXEC(@bucket_expr)
       AND f.TERM_GROUP     = tg.TERM_GROUP
       AND f.PROD_NAME_RES  = t.PROD_NAME_RES
       AND f.TSEGMENTNAME   = t.TSEGMENTNAME
       AND f.conv           = CAST(t.conv AS varchar(50))
WHERE   t.dt_rep = @Anchor
  AND   t.is_floatrate = 0
  AND   o.d_open = @Prev
  AND   o.d_open IS NOT NULL
  AND   f.BALANCE_GROUP IS NULL;

/* ── 3. рыночный средневзвешенный спред ──────────────────────────*/
SELECT
        BALANCE_GROUP, TERM_GROUP,
        PROD_NAME_RES, TSEGMENTNAME, conv,
        spread_mkt = SUM(out_rub*spread) / NULLIF(SUM(out_rub),0)
INTO    #mkt
FROM   (SELECT * FROM #fresh15 UNION ALL SELECT * FROM #fresh14) s
GROUP BY BALANCE_GROUP, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv;

/* ── 4. фиксы, закрывающиеся ≤ 31-08-25 ──────────────────────────*/
SELECT
        r.con_id,
        r.out_rub,
        r.rate_con,
        BALANCE_GROUP = EXEC(@bucket_expr),
        tg.TERM_GROUP,
        r.PROD_NAME_RES,
        r.TSEGMENTNAME,
        conv = CAST(r.conv AS varchar(50))
INTO    #roll_fix
FROM    ALM.ALM.vw_balance_rest_all r  WITH (NOLOCK)
CROSS  APPLY (SELECT TRY_CAST(r.dt_close AS date) d_close) c
LEFT JOIN WORK.man_TermGroup tg
          ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE   r.dt_rep = @Anchor
  AND   r.is_floatrate = 0
  AND   c.d_close <= @HorizonTo
  AND   c.d_close IS NOT NULL;

/* ── 5. матч + покрытие ──────────────────────────────────────────*/
SELECT  rf.* , m.spread_mkt
INTO    #match
FROM    #roll_fix rf
LEFT JOIN #mkt m
  ON  m.BALANCE_GROUP  = rf.BALANCE_GROUP
  AND m.TERM_GROUP     = rf.TERM_GROUP
  AND m.PROD_NAME_RES  = rf.PROD_NAME_RES
  AND m.TSEGMENTNAME   = rf.TSEGMENTNAME
  AND m.conv           = rf.conv;

/* ── 6. сводка ──────────────────────────────────────────────────*/
SELECT
    total_deals   = COUNT(*) ,
    covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals     = 100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
                    / COUNT(*) ,
    total_rub     = SUM(out_rub) ,
    covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub       = 100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
                    / NULLIF(SUM(out_rub),0);
                    
/* ── 7. детализация по BALANCE_GROUP × TERM_GROUP ───────────────*/
SELECT
    BALANCE_GROUP, TERM_GROUP,
    deals_tot = COUNT(*) ,
    deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals = 100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
                / COUNT(*) ,
    rub_tot   = SUM(out_rub) ,
    rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub   = 100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
                / NULLIF(SUM(out_rub),0)
FROM #match
GROUP BY BALANCE_GROUP, TERM_GROUP
ORDER BY pct_rub DESC;
```

**Что изменилось**

| Блок       | Изменение                                                                |
| ---------- | ------------------------------------------------------------------------ |
| 1 & 2      | `BALANCE_GROUP` рассчитывается через `CASE` вместо справочника.          |
| 4          | То же выражение используется для roll-over-фиксов.                       |
| `TRY_CAST` | Все `dt_open` / `dt_close` превращаются в `date` без ошибок конвертации. |

Скрипт выводит:

1. **Сводку** — доля договоров и объёма, для которых найден «рыночный» спред.
2. **Детализацию** по новым четырём бакетам × группам срока.

Копируйте-вставляйте целиком — переменные объявлены, типы согласованы.
