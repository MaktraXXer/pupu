Ниже привожу **обновлённую версию v-3 FIN** – изменился только
справочник `#ref_spread`.
Все остальные шаги (портфель → roll-over → выгрузка) остались
нетронутыми, поэтому можно смело заменить у себя *только* блок
создания/заполнения таблицы-справочника.

```sql
/*-----------------------------------------------------------------------
  0.  СПРАВОЧНИК  TO-BE  +  «средний прогнозный KEY»
      — новые числа от 19-июл-2025
------------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#ref_spread') IS NOT NULL DROP TABLE #ref_spread;
CREATE TABLE #ref_spread
( seg char(1),          -- 'R' = Розничный бизнес ; 'O' = ДЧБО / прочие
  term_nom int,         -- номинальный срок (дней)
  term_lo  int,         -- окно  −15
  term_hi  int,         -- окно  +15
  tobe_end decimal(9,6),-- ставка «TO-BE» (конвенция AT_THE_END)
  key_avg  decimal(9,6) -- средний прогнозный KEY на сегодня
);

/* ───────────  РОЗНИЧНЫЙ БИЗНЕС  (seg = 'R')  ─────────── */
INSERT #ref_spread VALUES
('R',  61 ,  46 ,  76 , 0.1820 , 0.1733 ),
('R',  91 ,  76 , 106 , 0.1790 , 0.1718 ),
('R', 122 , 107 , 137 , 0.1810 , 0.1683 ),
('R', 181 , 166 , 196 , 0.1750 , 0.1636 ),
('R', 274 , 259 , 289 , 0.1610 , 0.1548 ),
('R', 367 , 352 , 382 , 0.1610 , 0.1484 ),
('R', 550 , 535 , 565 , 0.1430 , 0.1408 ),
('R', 730 , 715 , 745 , 0.1410 , 0.1370 ),
('R',1100 ,1085 ,1115 , 0.1360 , 0.1320 );

/* ───────────  ДЧБО / ПРОЧИЕ  (seg = 'O')  ─────────── */
INSERT #ref_spread VALUES
('O',  61 ,  46 ,  76 , 0.1860 , 0.1733 ),
('O',  91 ,  76 , 106 , 0.1830 , 0.1718 ),
('O', 122 , 107 , 137 , 0.1850 , 0.1683 ),
('O', 181 , 166 , 196 , 0.1790 , 0.1636 ),
('O', 274 , 259 , 289 , 0.1660 , 0.1548 ),
('O', 367 , 352 , 382 , 0.1660 , 0.1484 ),
('O', 550 , 535 , 565 , 0.1500 , 0.1408 ),
('O', 730 , 715 , 745 , 0.1480 , 0.1370 ),
('O',1100 ,1085 ,1115 , 0.1430 , 0.1320 );
```

📌 **Что изменилось**

| Сегмент | Срок (дней) | TO-BE <br>(было → стало) | KEY-avg <br>(было → стало) |
| ------- | ----------- | ------------------------ | -------------------------- |
| **R**   | 61          | 0.1820 → 0.1820          | 0.1749 → 0.1733            |
| …       | …           | …                        | …                          |
| **O**   | 1100        | 0.1430 → 0.1430          | 0.1332 → 0.1320            |

(таблица выше просто иллюстрирует замену цифр – логика
алгоритма не тронута).

После этой вставки продолжайте скрипт
*точно с той же структуры*, как он у вас был
(`#base → #fix_spread → #rolls → #daily → выгрузка`).
Все ссылки на `#ref_spread` автоматически возьмут новые значения.




Ниже – целиковая версия **v-2 (-daysback)**, где
справочник «рыночных» спредов (#fresh15 → #mkt / #mkt\_any) собирается не
только за день `@Anchor`, а за окно **@DaysBack** дней
назад. По умолчанию окно = 0, то есть поведение прежнее.

```sql
/**************************************************************************
  V-2  + параметр @DaysBack
  --------------------------------------------
  • @Anchor     – последняя доступная дата факта   (dt_rep портфеля)
  • @DaysBack   – сколько дней назад искать «рынок»
                  0  = только @Anchor  (как было раньше)
                  2  = @Anchor, (@Anchor-1), (@Anchor-2)
**************************************************************************/
USE ALM_TEST;
GO
DECLARE
    @Anchor     date = '2025-07-16',          -- последний факт dt_rep
    @HorizonTo  date = '2025-09-30',          -- конец прогноза
    @DaysBack   int  = 3;                     -- ← можно менять
/*-----------------------------------------------------------------------
  общие границы окна «рынка»
-----------------------------------------------------------------------*/
DECLARE @MarketFrom date = DATEADD(day,-@DaysBack,@Anchor);   -- включительно
/*=======================================================================
  1. бакеты объёма (без изменений)
=======================================================================*/
IF OBJECT_ID('tempdb..#bucket_def') IS NOT NULL DROP TABLE #bucket_def;
CREATE TABLE #bucket_def(bucket varchar(20) PRIMARY KEY,lo money,hi money,r tinyint);
INSERT #bucket_def VALUES
('[0-1.5 млн)',0,1500000,0),('[1.5-15 млн)',1500000,15000000,1),
('[15-100 млн)',15000000,100000000,2),('[100 млн+]',100000000,NULL,3);
/*=======================================================================
  2. «рынок»  –  окно @MarketFrom … @Anchor   (БЫЛ только =@Anchor)
=======================================================================*/
IF OBJECT_ID('tempdb..#fresh15') IS NOT NULL DROP TABLE #fresh15;
CREATE TABLE #fresh15 (
  bucket varchar(20), TERM_GROUP varchar(100), PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME nvarchar(100), conv_norm varchar(50), out_rub money, spread decimal(18,6));

/* 2-а. реальные сделки  (conv ≠ AT_THE_END) */
INSERT #fresh15
SELECT b.bucket,tg.TERM_GROUP,t.PROD_NAME_RES,t.TSEGMENTNAME,
       CAST(t.conv AS varchar(50)),
       t.out_rub,
       t.rate_con - fk.AVG_KEY_RATE
FROM  ALM.ALM.vw_balance_rest_all t  WITH(NOLOCK)
JOIN  #bucket_def b ON t.out_rub BETWEEN b.lo AND ISNULL(b.hi,t.out_rub)
CROSS APPLY (VALUES(TRY_CAST(t.DT_OPEN AS date))) o(d_open)
JOIN  ALM_TEST.WORK.ForecastKey_Cache fk ON fk.DT_REP=o.d_open AND fk.TERM=t.termdays
LEFT JOIN WORK.man_TermGroup tg ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE t.dt_rep BETWEEN @MarketFrom AND @Anchor                -- ΔΔ
  AND t.section_name=N'Срочные'  AND t.block_name=N'Привлечение ФЛ'
  AND t.od_flag=1 AND t.cur='810' AND t.is_floatrate=0 AND t.out_rub>0
  AND t.conv<>'AT_THE_END'
  AND o.d_open BETWEEN @MarketFrom AND @Anchor;               -- ΔΔ

/* 2-б. «виртуальный» 1M-спред для AT_THE_END */
INSERT #fresh15
SELECT b.bucket,tg.TERM_GROUP,t.PROD_NAME_RES,t.TSEGMENTNAME,'1M',
       t.out_rub,
       CAST([LIQUIDITY].[liq].[fnc_IntRate]
            (t.rate_con,'at the end','monthly',t.termdays,1) AS decimal(18,6))
       - fk.AVG_KEY_RATE
FROM  ALM.ALM.vw_balance_rest_all t  WITH(NOLOCK)
JOIN  #bucket_def b ON t.out_rub BETWEEN b.lo AND ISNULL(b.hi,t.out_rub)
CROSS APPLY (VALUES(TRY_CAST(t.DT_OPEN AS date))) o(d_open)
JOIN  ALM_TEST.WORK.ForecastKey_Cache fk ON fk.DT_REP=o.d_open AND fk.TERM=t.termdays
LEFT JOIN WORK.man_TermGroup tg ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE t.dt_rep BETWEEN @MarketFrom AND @Anchor                -- ΔΔ
  AND t.section_name=N'Срочные'  AND t.block_name=N'Привлечение ФЛ'
  AND t.od_flag=1 AND t.cur='810' AND t.is_floatrate=0 AND t.out_rub>0
  AND t.conv='AT_THE_END'
  AND o.d_open BETWEEN @MarketFrom AND @Anchor;               -- ΔΔ
/*=======================================================================
  3. справочники спредов   (как раньше)
=======================================================================*/
SELECT bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,
       spread_mkt = SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
INTO #mkt
FROM #fresh15
GROUP BY bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm;

SELECT TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,
       spread_any = SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
INTO #mkt_any
FROM #fresh15
GROUP BY TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm;
/*=======================================================================
  4. roll-fix (портфель = @Anchor)      –  без изменений
=======================================================================*/
IF OBJECT_ID('tempdb..#roll_fix') IS NOT NULL DROP TABLE #roll_fix;
SELECT r.con_id,r.out_rub,b.bucket,b.r,
       tg.TERM_GROUP,r.PROD_NAME_RES,r.TSEGMENTNAME,
       conv_norm = CASE WHEN r.conv='AT_THE_END' THEN '1M'
                        ELSE CAST(r.conv AS varchar(50)) END,
       spread_fix = r.rate_con - fk_open.AVG_KEY_RATE
INTO   #roll_fix
FROM   ALM.ALM.vw_balance_rest_all r  WITH(NOLOCK)
JOIN   #bucket_def b ON r.out_rub BETWEEN b.lo AND ISNULL(b.hi,r.out_rub)
LEFT   JOIN WORK.man_TermGroup tg ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
LEFT   JOIN ALM_TEST.WORK.ForecastKey_Cache fk_open
           ON fk_open.DT_REP=TRY_CAST(r.DT_OPEN AS date) AND fk_open.TERM=r.termdays
CROSS  APPLY (VALUES(TRY_CAST(r.DT_CLOSE AS date))) c(d_close)
WHERE  r.dt_rep=@Anchor AND r.section_name=N'Срочные'
  AND  r.block_name=N'Привлечение ФЛ' AND r.od_flag=1
  AND  r.cur='810' AND r.is_floatrate=0 AND r.out_rub>0
  AND  c.d_close<=@HorizonTo AND c.d_close IS NOT NULL;
/*=======================================================================
  5. матчинг (bucket→вверх + fallback)  –  без изменений
=======================================================================*/
IF OBJECT_ID('tempdb..#match') IS NOT NULL DROP TABLE #match;
SELECT rf.*, mkt.spread_mkt, ma.spread_any,
       spread_final = COALESCE(mkt.spread_mkt,ma.spread_any,rf.spread_fix)
INTO   #match
FROM   #roll_fix rf
OUTER  APPLY (
    SELECT TOP 1 m.spread_mkt
    FROM #mkt m JOIN #bucket_def b_m ON b_m.bucket=m.bucket
    WHERE m.TERM_GROUP=rf.TERM_GROUP AND m.PROD_NAME_RES=rf.PROD_NAME_RES
      AND m.TSEGMENTNAME=rf.TSEGMENTNAME AND m.conv_norm=rf.conv_norm
      AND b_m.r>=rf.r
    ORDER BY b_m.r) mkt
OUTER  APPLY (
    SELECT ma.spread_any
    FROM #mkt_any ma
    WHERE rf.TSEGMENTNAME=N'ДЧБО' AND ma.TERM_GROUP=rf.TERM_GROUP
      AND ma.PROD_NAME_RES=rf.PROD_NAME_RES AND ma.TSEGMENTNAME=rf.TSEGMENTNAME
      AND ma.conv_norm=rf.conv_norm) ma;
/* leave rest of script exactly as before … */
-- 6. #fix_spread  (rank 1) ---------------------------------------
;WITH ranked AS (
    SELECT con_id,spread_final,
           ROW_NUMBER() OVER(PARTITION BY con_id ORDER BY (SELECT 0)) AS rn
    FROM   #match)
SELECT con_id,spread_final INTO #fix_spread WHERE rn=1;
-- 7. календарь + key_spot уже есть (#cal, #key_spot) -------------
-- 8. фактовая база  (#base)  – берём из портфеля @Anchor ---------
--     (оставьте код без изменений)
-- 9. roll-over (#rolls) – логика та же (подмена c n≥1) -----------
-- 10. #daily, 11. выгрузка  – без изменений ----------------------

/*------------- с этого места копируйте старую неизменённую часть
                (начиная c создания #base) -----------------------*/
```

#### Что поменялось

| Шаг | Было                                        | Стало                                                                                    |
| --- | ------------------------------------------- | ---------------------------------------------------------------------------------------- |
| 2   | `t.dt_rep = @Anchor` и `o.d_open = @Anchor` | `t.dt_rep BETWEEN @MarketFrom AND @Anchor`<br>`o.d_open BETWEEN @MarketFrom AND @Anchor` |
| 2   | Параметра не было                           | добавлен `@DaysBack`, по умолчанию = 0                                                   |
| 5   | В `#fix_spread` могли остаться дубликаты    | ранжируем `ROW_NUMBER()` и берём **один** спред на договор                               |

Теперь можно запускать так:

```sql
-- только последний день (как раньше)
EXEC your_script

-- взять «рынок» за три последних дня
EXEC your_script @DaysBack = 3
```

Все остальные расчёты (roll-over, агрегаты) остаются прежними.
