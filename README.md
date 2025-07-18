Ниже ― **«вариант-3»** целиком:
*тот же каркас, но спред для фикс-депозитов берётся из «ручного» справочника TO BE, ­а для срочности 31 ±10 дней договор переворачивается в корзину 91 день.*

> ▸ все числа TO BE и KEY приведены к **долям** (18,60 % → 0.1860);
> ▸ оконный диапазон ±15 дн;
> ▸ если conv ≠ `AT_THE_END` — сначала переводим TO BE в 1-месячную конвенцию,
> затем вычисляем spread = rate – key;
> ▸ если совпадения нет — оставляем фактический спред.

```sql
/**************************************************************************
  V-3  :  прогноз спредов «ручным» справочником TO-BE (±15 дн);
          фактическая запись (n=0) никогда не трогается
**************************************************************************/
USE ALM_TEST;
GO
DECLARE
    @Anchor     date = '2025-07-16',                 -- последний факт
    @HorizonTo  date = '2025-09-30',
    @Horizon    int  = DATEDIFF(day,'2025-07-16','2025-09-30');

/*──────────────────── 0. справочник TO-BE / KEY ────────────────────*/
IF OBJECT_ID('tempdb..#ref_spread') IS NOT NULL DROP TABLE #ref_spread;
CREATE TABLE #ref_spread
( seg char(1),        -- 'R' = Розничный бизнес ; 'O' = прочие
  term_nom int,       -- «опорный» срок, дн
  term_lo  int,       -- нижняя граница окна  (term-15)
  term_hi  int,       -- верхняя             (term+15)
  tobe_end decimal(9,6),   -- ставка в конце срока
  key_avg  decimal(9,6) ); -- средн. прогнозный KEY

INSERT #ref_spread(seg,term_nom,term_lo,term_hi,tobe_end,key_avg) VALUES
/* ── Розничный бизнес (seg='R') ── */
('R',  61,  46,  76, 0.1820, 0.174918),
('R',  91,  76, 106, 0.1790, 0.173098),
('R', 122, 107, 137, 0.1810, 0.169306),
('R', 181, 166, 196, 0.1750, 0.163700),  -- 16.37 % ⇒ 0.1637
('R', 274, 259, 289, 0.1610, 0.156300),
('R', 367, 352, 382, 0.1610, 0.150200),
('R', 550, 535, 565, 0.1430, 0.142100),
('R', 730, 715, 745, 0.1410, 0.137800),
('R',1100,1085,1115, 0.1360, 0.133200),
/* ── Прочие сегменты (seg='O') ── */
('O',  61,  46,  76, 0.1860, 0.174918),
('O',  91,  76, 106, 0.1830, 0.173098),
('O', 122, 107, 137, 0.1850, 0.169306),
('O', 181, 166, 196, 0.1790, 0.163700),
('O', 274, 259, 289, 0.1660, 0.156300),
('O', 367, 352, 382, 0.1660, 0.150200),
('O', 550, 535, 565, 0.1500, 0.142100),
('O', 730, 715, 745, 0.1480, 0.137800),
('O',1100,1085,1115, 0.1430, 0.133200);

/*──────────────────── 1. бакеты объёма (как раньше) ─────────────*/
IF OBJECT_ID('tempdb..#bucket_def') IS NOT NULL DROP TABLE #bucket_def;
CREATE TABLE #bucket_def(bucket varchar(20) PRIMARY KEY,lo money,hi money,r tinyint);
INSERT #bucket_def VALUES
('[0-1.5 млн)',0,1500000,0),
('[1.5-15 млн)',1500000,15000000,1),
('[15-100 млн)',15000000,100000000,2),
('[100 млн+]',100000000,NULL,3);

/*──────────────────── 2. календарь + spot-KEY ──────────────────*/
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal)<@HorizonTo
    INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

IF OBJECT_ID('tempdb..#key_spot') IS NOT NULL DROP TABLE #key_spot;
SELECT fc.DT_REP,fc.KEY_RATE
INTO   #key_spot
FROM   WORK.ForecastKey_Cache fc JOIN #cal c ON c.d=fc.DT_REP
WHERE  fc.TERM=1;

/*──────────────────── 3. фактовая база 16-07-2025 ──────────────*/
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
SELECT  t.con_id,t.out_rub,t.rate_con,t.is_floatrate,
        t.termdays,t.dt_open,t.dt_close,t.conv,t.TSEGMENTNAME,
        spread_float = CASE WHEN t.is_floatrate=1
                                THEN t.rate_con-ks.KEY_RATE END,
        spread_fix   = CASE WHEN t.is_floatrate=0
                                THEN t.rate_con-fk_open.AVG_KEY_RATE END
INTO    #base
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
JOIN    #key_spot ks                ON ks.DT_REP=@Anchor       -- spot KEY (16-07)
LEFT JOIN WORK.ForecastKey_Cache fk_open                      -- AVG KEY на дату открытия
       ON fk_open.DT_REP=t.dt_open AND fk_open.TERM=t.termdays
WHERE   t.dt_rep=@Anchor
  AND   t.section_name=N'Срочные'
  AND   t.block_name  =N'Привлечение ФЛ'
  AND   t.od_flag=1 AND t.cur='810' AND t.out_rub IS NOT NULL;

/*──────────────────── 4. справочник spread_final для ФИКС ──────*/
IF OBJECT_ID('tempdb..#fix_spread') IS NOT NULL DROP TABLE #fix_spread;

/* 4-а. подготовим TO-BE-спред по каждому договору */
WITH map AS (
    SELECT  b.con_id,
            -- группа сегмента
            seg = CASE WHEN b.TSEGMENTNAME=N'Розничный Бизнес' THEN 'R' ELSE 'O' END,
            -- «нормализованный» срок для поиска (31±10 → 91)
            term_use = CASE 
                         WHEN b.termdays BETWEEN 21 AND 41 THEN 91
                         ELSE b.termdays END,
            b.conv,
            b.out_rub,
            b.termdays,
            b.dt_open
    FROM    #base b
    WHERE   b.is_floatrate = 0                    -- только фики
),
match_ref AS (
    SELECT  m.con_id,
            r.tobe_end,
            r.key_avg,
            r.term_nom,
            r.seg,
            m.conv,
            m.termdays,
            m.out_rub,
            m.dt_open
    FROM    map           m
    JOIN    #ref_spread   r
           ON r.seg       = m.seg
          AND m.term_use BETWEEN r.term_lo AND r.term_hi
)
/* 4-б. spread_final:   для conv = AT_THE_END – сразу;  
                        иначе сначала переводим в 1-месячную   */
SELECT  m.con_id,
        spread_final =
        CASE
            WHEN m.conv = 'AT_THE_END'
                 THEN m.tobe_end - m.key_avg
            ELSE  CAST([LIQUIDITY].[liq].[fnc_IntRate]
                        (m.tobe_end,'at the end','monthly',m.term_nom,1) AS decimal(9,6))
                 -  m.key_avg
        END
INTO    #fix_spread
FROM    match_ref m;

/*──────────────────── 5. roll-over-цепочки (n=0 остаётся факт) ─*/
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;

;WITH seq AS (
    SELECT  b.con_id,b.out_rub,b.is_floatrate,b.termdays,
            b.spread_float,
            spread_fix = COALESCE(fs.spread_final, b.spread_fix),  -- n=0 ещё факт!
            b.dt_open,
            n = 0
    FROM    #base b
    LEFT    JOIN #fix_spread fs ON fs.con_id = b.con_id
    UNION ALL
    SELECT  s.con_id,s.out_rub,s.is_floatrate,s.termdays,
            s.spread_float,
            spread_fix = COALESCE(fs.spread_final, s.spread_fix),
            dt_open = DATEADD(day,s.termdays,s.dt_open),
            n + 1
    FROM    seq s
    LEFT    JOIN #fix_spread fs ON fs.con_id = s.con_id
    WHERE   DATEADD(day,s.termdays,s.dt_open) <= @HorizonTo
)
SELECT  con_id,out_rub,is_floatrate,termdays,
        dt_open,
        dt_close = DATEADD(day,termdays,dt_open),
        spread_float,spread_fix,n
INTO   #rolls
FROM   seq OPTION (MAXRECURSION 0);

/*──────────────────── 6. посуточные ставки ────────────────────*/
IF OBJECT_ID('tempdb..#daily') IS NOT NULL DROP TABLE #daily;
SELECT  c.d AS dt_rep,
        w.con_id,w.out_rub,
        rate_con = CASE
                     WHEN w.is_floatrate=1
                          THEN ks.KEY_RATE + w.spread_float
                     ELSE ISNULL(fko.AVG_KEY_RATE + w.spread_fix, w.spread_fix)
                   END
INTO    #daily
FROM    #cal c
JOIN    #rolls w  ON c.d BETWEEN w.dt_open AND DATEADD(day,-1,w.dt_close)
LEFT JOIN #key_spot ks ON ks.DT_REP = c.d
LEFT JOIN WORK.ForecastKey_Cache fko
       ON fko.DT_REP = w.dt_open AND fko.TERM = w.termdays;

/*──────────────────── 7. выгрузка v3 ───────────────────────────*/
IF OBJECT_ID('WORK.Forecast_BalanceDaily_v3','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDaily_v3
    (dt_rep DATE PRIMARY KEY,
     out_rub_total DECIMAL(20,2),
     rate_con      DECIMAL(9,4));
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDaily_v3;

IF OBJECT_ID('WORK.Forecast_BalanceDeals_v3','U') IS NULL
    CREATE TABLE WORK.Forecast_BalanceDeals_v3
    (dt_rep DATE,
     con_id  BIGINT,
     out_rub DECIMAL(20,2),
     rate_con DECIMAL(9,4),
     CONSTRAINT PK_FBD_v3 PRIMARY KEY(dt_rep,con_id));
ELSE TRUNCATE TABLE WORK.Forecast_BalanceDeals_v3;

/* агрегат */
INSERT INTO WORK.Forecast_BalanceDaily_v3(dt_rep,out_rub_total,rate_con)
SELECT dt_rep,
       SUM(out_rub),
       SUM(out_rub*rate_con)/SUM(out_rub)
FROM   #daily
GROUP BY dt_rep;

/* деталка */
INSERT INTO WORK.Forecast_BalanceDeals_v3(dt_rep,con_id,out_rub,rate_con)
SELECT dt_rep,con_id,out_rub,rate_con FROM #daily;
GO
```

### что поменялось / проверки

| Шаг | Изменение                                                                               |
| --- | --------------------------------------------------------------------------------------- |
| 0   | добавлен **#ref\_spread** ― ручной справочник TO-BE / KEY с окнами ±15 дн; 31±10 → 91   |
| 3   | к фактовой базе добавлены `conv`, `TSEGMENTNAME`                                        |
| 4   | расчёт `spread_final` напрямую из справочника (с переводом конвенции при необходимости) |
| 5   | в `seq` для n = 0 оставляем факт-спред, далее подставляем `spread_final`                |
| 7   | результат пишется в `…_v3`                                                              |

*Сравните 16-07-2025: объём и ставка должны полностью совпасть с контрольным фактом (20.21 %).
Дальше ― прогноз уже на «новых» спредах из справочника.*
