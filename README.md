### Что значит «сопоставить»

* **Комбинация признаков** (5-мерный ключ):

```
(balance_group, term_group, PROD_NAME_RES, TSEGMENTNAME, conv)
```

* **«Свежие» открытые** фикс-депозиты — сначала берём **15 июля**, затем,
  для недостающих комбинаций, добираем **14 июля**.
* Для каждой найденной комбинации рассчитываем **средневзвешенный** (по
  объёму) спред — это и есть «рыночный» spread.

### Чем отличается от «спред = старый, что был»

В предыдущей логике rollover-депозит наследовал **свой** исторический
спред. Теперь он получает **рыночный** спред (из свежих открытий) для
той же 5-мерной категории; если такого нет — остаётся со старым.

---

## T-SQL: 1) делаем таблицу рыночных спредов, 2) матчим

```sql
/* параметры */
DECLARE @Anchor date = '2025-07-15',
        @Prev   date = DATEADD(day,-1,@Anchor),  -- 14-07-25
        @HorizonTo date = '2025-08-31';

/* ───────────────── 1. «свежие» фикс-депозиты 15-го ─────────────── */
IF OBJECT_ID('tempdb..#fresh_15') IS NOT NULL DROP TABLE #fresh_15;

SELECT  bg.BALANCE_GROUP,
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,
        t.conv,
        t.out_rub,
        spread = t.rate_con - fk.AVG_KEY_RATE
INTO    #fresh_15
FROM    ALM.ALM.vw_balance_rest_all t
JOIN    WORK.ForecastKey_Cache fk
           ON fk.DT_REP = t.dt_open
          AND fk.TERM   = t.termdays
LEFT JOIN WORK.man_BalanceGroup bg
           ON t.out_rub BETWEEN bg.BALANCE_FROM AND bg.BALANCE_TO
LEFT JOIN WORK.man_TermGroup tg
           ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE   t.dt_rep      = @Anchor
  AND   t.is_floatrate = 0
  AND   t.dt_open      = @Anchor;          -- открыты 15-го

/* ─────────────── 2. резерв 14-го, только не перекрытые  ────────── */
IF OBJECT_ID('tempdb..#fresh_14') IS NOT NULL DROP TABLE #fresh_14;

SELECT  bg.BALANCE_GROUP,
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,
        t.conv,
        t.out_rub,
        spread = t.rate_con - fk.AVG_KEY_RATE
INTO    #fresh_14
FROM    ALM.ALM.vw_balance_rest_all t
JOIN    WORK.ForecastKey_Cache fk
           ON fk.DT_REP = t.dt_open
          AND fk.TERM   = t.termdays
LEFT JOIN WORK.man_BalanceGroup bg
           ON t.out_rub BETWEEN bg.BALANCE_FROM AND bg.BALANCE_TO
LEFT JOIN WORK.man_TermGroup tg
           ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
LEFT JOIN #fresh_15 f
       ON  f.BALANCE_GROUP  = bg.BALANCE_GROUP
       AND f.TERM_GROUP     = tg.TERM_GROUP
       AND f.PROD_NAME_RES  = t.PROD_NAME_RES
       AND f.TSEGMENTNAME   = t.TSEGMENTNAME
       AND f.conv           = t.conv
WHERE   t.dt_rep      = @Anchor
  AND   t.is_floatrate = 0
  AND   t.dt_open      = @Prev           -- открыты 14-го
  AND   f.BALANCE_GROUP IS NULL;         -- нет аналога в 15-м

/* ─────────────── 3. итоговые «рыночные» спреды ────────────────── */
IF OBJECT_ID('tempdb..#market_spread') IS NOT NULL DROP TABLE #market_spread;

SELECT  BALANCE_GROUP, TERM_GROUP,
        PROD_NAME_RES, TSEGMENTNAME, conv,
        spread_mkt = SUM(out_rub*spread)/SUM(out_rub)  -- взвешенное
INTO    #market_spread
FROM   (
        SELECT * FROM #fresh_15
        UNION ALL
        SELECT * FROM #fresh_14
       ) x
GROUP BY BALANCE_GROUP, TERM_GROUP,
         PROD_NAME_RES, TSEGMENTNAME, conv;

/* ─────────── 4. фикс-депозиты, закрывающиеся ≤ 31-08 ──────────── */
IF OBJECT_ID('tempdb..#roll_fix') IS NOT NULL DROP TABLE #roll_fix;

SELECT  t.con_id, t.out_rub, t.rate_con,
        bg.BALANCE_GROUP, tg.TERM_GROUP,
        t.PROD_NAME_RES, t.TSEGMENTNAME, t.conv,
        t.termdays, t.dt_open, t.dt_close
INTO    #roll_fix
FROM    ALM.ALM.vw_balance_rest_all t
LEFT JOIN WORK.man_BalanceGroup bg
         ON t.out_rub BETWEEN bg.BALANCE_FROM AND bg.BALANCE_TO
LEFT JOIN WORK.man_TermGroup tg
         ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE   t.dt_rep      = @Anchor
  AND   t.is_floatrate = 0
  AND   t.dt_close     <= @HorizonTo;

/* ─────────── 5. матчинг + оценка покрытия ─────────────────────── */
SELECT  r.*,
        m.spread_mkt
INTO    #roll_match
FROM    #roll_fix r
LEFT JOIN #market_spread m
  ON  m.BALANCE_GROUP  = r.BALANCE_GROUP
  AND m.TERM_GROUP     = r.TERM_GROUP
  AND m.PROD_NAME_RES  = r.PROD_NAME_RES
  AND m.TSEGMENTNAME   = r.TSEGMENTNAME
  AND m.conv           = r.conv;

/*  общий охват  */
SELECT  total_deals   = COUNT(*),
        covered_deals = COUNT(spread_mkt),
        pct_deals     = 100.0*COUNT(spread_mkt)/COUNT(*),
        total_rub     = SUM(out_rub),
        covered_rub   = SUM(CASE WHEN spread_mkt IS NULL THEN 0 ELSE out_rub END),
        pct_rub       = 100.0*SUM(CASE WHEN spread_mkt IS NULL THEN 0 ELSE out_rub END)
/                                  /NULLIF(SUM(out_rub),0)
FROM    #roll_match;

/* деталка по группам  (пригодится для анализа) */
SELECT  BALANCE_GROUP, TERM_GROUP,
        deals_tot = COUNT(*),
        deals_ok  = COUNT(spread_mkt),
        pct_deals = 100.0*COUNT(spread_mkt)/COUNT(*),
        rub_tot   = SUM(out_rub),
        rub_ok    = SUM(CASE WHEN spread_mkt IS NULL THEN 0 ELSE out_rub END),
        pct_rub   = 100.0*SUM(CASE WHEN spread_mkt IS NULL THEN 0 ELSE out_rub END)
/                               /NULLIF(SUM(out_rub),0)
FROM    #roll_match
GROUP BY BALANCE_GROUP, TERM_GROUP
ORDER BY pct_rub DESC;
```

### Что делает скрипт

1. **Шаг 1–2** — собирает «рыночные» спреды:

   * сначала 15 июля (`#fresh_15`),
   * комбиниции, которых нет — добирает из 14 июля (`#fresh_14`).
     Усредняет по объёму → `#market_spread`.

2. **Шаг 4** — отбирает фиксы, закрывающиеся до 31-08 (roll-over).

3. **Шаг 5** — джойнит roll-over-фиксы с `#market_spread`
   (полное совпадение пятёрки атрибутов) и считает охват
   как по количеству, так и по сумме.

После просмотра покрытие ≥ 80 % ⇒ можно внедрять «рыночный»
спред; отклонение *контролируем* (детализация покажет,
какие группы не «матч‐нуты»).
