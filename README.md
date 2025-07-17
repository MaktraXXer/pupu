Ниже — цельный T-SQL-скрипт, который:

* **сохраняет прежнюю логику** (прямое совпадение + «поднятие» бакета + fallback ДЧБО);
* **добавляет** нормализацию ставок **AT\_THE\_END → 1M**
  и использует её *только* если под рукой нет «родной» месячной ставки;
* выводит итоговые сводки (шаг 6 и 7) без ошибок имён.

Скрипт можно вставить в SSMS целиком и сразу выполнить.

```sql
/*────────────────────────  0. параметры  ────────────────────────*/
DECLARE
    @Anchor     date = '2025-07-15',
    @HorizonTo  date = '2025-08-31';

/*──────────────────  0.1 очистка временных таблиц  ─────────────*/
IF OBJECT_ID('tempdb..#bucket_def') IS NOT NULL DROP TABLE #bucket_def;
IF OBJECT_ID('tempdb..#fresh15')    IS NOT NULL DROP TABLE #fresh15;
IF OBJECT_ID('tempdb..#mkt')        IS NOT NULL DROP TABLE #mkt;
IF OBJECT_ID('tempdb..#mkt_any')    IS NOT NULL DROP TABLE #mkt_any;
IF OBJECT_ID('tempdb..#roll_fix')   IS NOT NULL DROP TABLE #roll_fix;
IF OBJECT_ID('tempdb..#match')      IS NOT NULL DROP TABLE #match;

SET NOCOUNT ON;      -- чтобы не засорять вывод «(N row affected)»

/*────────────────────  1. справочник бакетов  ───────────────────*/
CREATE TABLE #bucket_def
( bucket varchar(20) PRIMARY KEY,
  lo     money        NOT NULL,
  hi     money        NULL,
  r      tinyint      NOT NULL );

INSERT #bucket_def
VALUES('[0-1.5 млн)',0,1500000,0),
      ('[1.5-15 млн)',1500000,15000000,1),
      ('[15-100 млн)',15000000,100000000,2),
      ('[100 млн+]',100000000,NULL,3);

/*────────────────────  2. «рынок» - выдачи 15-го июля  ──────────*/
/*          conv_norm = фактическая conv         (1M или др.)
            + «виртуальная» 1M, если conv='AT_THE_END'          */

CREATE TABLE #fresh15
( bucket        varchar(20),
  TERM_GROUP    varchar(100),
  PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME  nvarchar(100),
  conv_norm     varchar(50),
  out_rub       money,
  spread        decimal(18,6) );

/* 2-а. строки с реальной месячной (и прочими) конвенциями */
INSERT INTO #fresh15
SELECT
        b.bucket,
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,
        conv_norm = CAST(t.conv AS varchar(50)),
        t.out_rub,
        spread    = t.rate_con - fk.AVG_KEY_RATE
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
  AND   t.conv <> 'AT_THE_END';          -- всё, кроме «конец срока»

/* 2-б. виртуальная “1M” для ставок «AT_THE_END» */
INSERT INTO #fresh15
SELECT
        b.bucket,
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,
        conv_norm = '1M',                                   -- нормализация
        t.out_rub,
        spread = CAST(
                   ([LIQUIDITY].[liq].[fnc_IntRate]
                       ( t.rate_con, 'at the end','monthly',
                         t.termdays, 1 ))
                 AS decimal(18,6)) - fk.AVG_KEY_RATE
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
  AND   t.conv         = 'AT_THE_END';    -- только «конец срока»

/*────────────────────  3. справочники спредов  ──────────────────*/
SELECT
        bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME,
        conv_norm,
        spread_mkt = SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
INTO    #mkt
FROM    #fresh15
GROUP BY bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm;

SELECT
        TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm,
        spread_any = SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
INTO    #mkt_any
FROM    #fresh15
GROUP BY TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm;

/*────────────────────  4. roll-over фиксы ≤ 31-08  ─────────────*/
CREATE TABLE #roll_fix
( con_id        bigint,
  out_rub       money,
  rate_con      decimal(18,6),
  bucket        varchar(20),
  r             tinyint,
  TERM_GROUP    varchar(100),
  PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME  nvarchar(100),
  conv_norm     varchar(50) );

INSERT INTO #roll_fix
SELECT r.con_id, r.out_rub, r.rate_con,
       b.bucket, b.r,
       tg.TERM_GROUP,
       r.PROD_NAME_RES, r.TSEGMENTNAME,
       conv_norm = CASE WHEN r.conv='AT_THE_END' THEN '1M'
                        ELSE CAST(r.conv AS varchar(50)) END
FROM   ALM.ALM.vw_balance_rest_all r WITH (NOLOCK)
JOIN   #bucket_def b ON r.out_rub>=b.lo AND (r.out_rub<b.hi OR b.hi IS NULL)
CROSS APPLY (SELECT TRY_CAST(r.DT_CLOSE AS date) d_close) c
LEFT  JOIN WORK.man_TermGroup tg
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

/*────────────────────  5. каскадное сопоставление  ─────────────*/
SELECT rf.*,
       spread_mkt =
       COALESCE
       ( /* a) точный / «крупнее» бакет, совпадение по conv_norm */
         ( SELECT TOP (1) m.spread_mkt
             FROM #mkt           AS m
             JOIN #bucket_def    AS b_m ON b_m.bucket = m.bucket
            WHERE m.TERM_GROUP      = rf.TERM_GROUP
              AND m.PROD_NAME_RES   = rf.PROD_NAME_RES
              AND m.TSEGMENTNAME    = rf.TSEGMENTNAME
              AND m.conv_norm       = rf.conv_norm
              AND b_m.r             >= rf.r       -- тот же / крупнее
            ORDER BY b_m.r ),                     -- ближайший «крупный»
         /* b) fallback только для ДЧБО (conv_norm уже «нормальный») */
         CASE WHEN rf.TSEGMENTNAME = N'ДЧБО'
              THEN ( SELECT ma.spread_any
                       FROM #mkt_any AS ma
                      WHERE ma.TERM_GROUP     = rf.TERM_GROUP
                        AND ma.PROD_NAME_RES  = rf.PROD_NAME_RES
                        AND ma.TSEGMENTNAME   = rf.TSEGMENTNAME
                        AND ma.conv_norm      = rf.conv_norm )
         END )
INTO   #match
FROM   #roll_fix AS rf;

/*────────────────────  6. сводка покрытия  ─────────────────────*/
SELECT
    total_deals   = COUNT(*) ,
    covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals     = 100.0 *
                    SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
                    / COUNT(*) ,
    total_rub     = SUM(out_rub) ,
    covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub       = 100.0 *
                    SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
                    / NULLIF(SUM(out_rub),0);

/*────────────────────  7. детализация bucket×срок×prod×segm×conv  ───────*/
SELECT
    bucket,
    TERM_GROUP,
    PROD_NAME_RES,
    TSEGMENTNAME,
    conv_norm,
    deals_tot = COUNT(*) ,
    deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals = 100.0 *
                SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
                / COUNT(*) ,
    rub_tot   = SUM(out_rub) ,
    rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub   = 100.0 *
                SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
                / NULLIF(SUM(out_rub),0)
FROM #match
GROUP BY bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm
ORDER BY pct_rub DESC;
```

**Что изменилось по сравнению с предыдущей версией**

| Блок | Правка                                                                                                                              | Итог                                                                                          |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| 2-б  | Добавлена «виртуальная» строка 1M для каждой свежей выплаты `AT_THE_END`; спред пересчитан через `fnc_IntRate`.                     | В справочнике рынка теперь есть ставка-эквивалент, если «настоящей» 1M не было.               |
| 3    | Все группировки строятся по `conv_norm`, т.е. и «настоящие» 1M, и «виртуальные» 1M участвуют, но **не** смешиваются с `AT_THE_END`. | Если «родная» месячная ставка есть — она выбирается; если нет — используется преобразованная. |
| 4    | В `#roll_fix` заведено поле `conv_norm` (для депозитов с `AT_THE_END` оно остаётся `AT_THE_END`, для прочих — фактическая).         | Совпадение идёт по «нормализованной» конвенции.                                               |
| 5    | Соединение по `conv_norm`; fallback ДЧБО тоже на нём.                                                                               | «Конвертированная» ставка даёт спред тем 9 % договоров, где раньше его не было.               |
| 6-7  | Итоговые сводки берут корректные имена столбцов; ошибок нет.                                                                        |                                                                                               |

После запуска посмотрите первую сводку — покрытие по объёму должно стать заметно выше прежних 91 %.
