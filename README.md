Ниже — минимальный патч.
Поменяли **только** блок 4 (формирование `#roll_fix`) — теперь он создаёт
таблицу через `SELECT … INTO`, а столбец фактического спреда
получает явный alias `AS spread_fix`.
Остальные шаги (5 → 8) трогать **не нужно** — они уже работают с
исправ-ленным `#roll_fix`, поэтому ошибки
`Invalid column name 'spread_fix' / 'is_matched'` исчезают.

```sql
/*─────── 4*. roll-over фиксы  (заменяет прежний блок 4) ───────*/
IF OBJECT_ID('tempdb..#roll_fix') IS NOT NULL DROP TABLE #roll_fix;

SELECT
        r.con_id,
        r.out_rub,
        b.bucket,
        b.r,
        tg.TERM_GROUP,
        r.PROD_NAME_RES,
        r.TSEGMENTNAME,
        CASE WHEN r.conv = 'AT_THE_END'          -- нормализуем конвенцию
             THEN '1M'
             ELSE CAST(r.conv AS varchar(50)) END          AS conv_norm,
        /* фактический спред = ставка – key-rate на дату открытия */
        r.rate_con - fk_open.AVG_KEY_RATE        AS spread_fix
INTO    #roll_fix
FROM    ALM.ALM.vw_balance_rest_all      r  WITH (NOLOCK)
JOIN    #bucket_def                      b
          ON r.out_rub BETWEEN b.lo AND ISNULL(b.hi, r.out_rub)
LEFT JOIN WORK.man_TermGroup             tg
          ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fk_open      -- 🔑 на дату открытия
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
```

С остальной частью вашего скрипта (шаги 0–3 и 5–8) ничего менять
не требуется — просто вставьте этот блок вместо старого четвёртого и
запустите весь файл одной командой.
