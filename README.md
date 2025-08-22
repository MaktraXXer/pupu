Коротко: у тебя «ролловер» стартует в тот же день, когда старый депозит закрывается (dt_open(next)=dt_close(prev)), а в факте на 2025-08-20 новые договоры, как правило, появляются со следующего дня (D+1). Плюс ты исключаешь сам день закрытия из старого договора (… BETWEEN dt_open AND dt_close-1). В результате на 20-е число в прогнозе ты уже учитываешь новый (O/R) депозит, а в факте — ещё нет, и средневзвешенная ставка расходится.

Что именно сейчас не совпадает с фактом
	1.	Старые сделки с dt_close='2025-08-20' в твоём #daily не попадают на 20-е (из-за dt_close-1).
	2.	Новые сделки после ролловера попадают уже 20-го (из-за dt_open(next)=dt_close(prev)).
	3.	В ALM.ALM.vw_balance_rest_all на 2025-08-20 обычно нет уже «новых» срочных, они становятся активными на 2025-08-21 (и потому средневзвешенная ниже/выше твоей модели).

Как поправить (минимальные правки)

Сделать день закрытия включительным для старого депозита и переносить дату открытия ролловера на D+1. Тогда 20-е число совпадёт с фактом.

Замени в блоках 5–6 вот так:

5. ROLL-OVER-цепочки: (старт следующего цикла = dt_close + 1)

/*====================================================================
     5.  ROLL-OVER-цепочки (n = 0,1,2,…) — next dt_open = dt_close + 1
  ====================================================================*/
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;
;WITH seq AS (
    SELECT con_id,out_rub,is_floatrate,termdays,dt_open,
           spread_float,
           spread_fix = spread_fix_fact,     -- факт для n=0
           spread_final,
           n = 0
    FROM   #base
    UNION ALL
    SELECT s.con_id,s.out_rub,s.is_floatrate,s.termdays,
           -- ⬇️ раньше было: DATEADD(day,s.termdays,s.dt_open)
           DATEADD(day, s.termdays + 1, s.dt_open) AS dt_open,  -- старт с D+1
           s.spread_float,
           s.spread_final,                -- для n>=1 используем справочник
           s.spread_final,
           n + 1
    FROM   seq s
    WHERE  DATEADD(day, s.termdays, s.dt_open) <= @HorizonTo
)
SELECT
       con_id, out_rub, is_floatrate, termdays,
       dt_open,
       dt_close = DATEADD(day, termdays, dt_open),  -- dt_close фактический
       spread_float, spread_fix
INTO   #rolls
FROM   seq
OPTION (MAXRECURSION 0);

6. ПОСУТОЧНЫЕ СТАВКИ: (делаем день закрытия включительно)

/*====================================================================
     6.  ПОСУТОЧНЫЕ СТАВКИ и АГРЕГАТ — день закрытия включительно
  ====================================================================*/
IF OBJECT_ID('tempdb..#daily') IS NOT NULL DROP TABLE #daily;
SELECT c.d AS dt_rep,
       r.con_id,
       r.out_rub,
       rate_con = CASE
                     WHEN r.is_floatrate = 1
                          THEN ks.KEY_RATE + r.spread_float         -- FLOAT = spot + спред
                     ELSE ISNULL(fk.AVG_KEY_RATE + r.spread_fix,    -- FIX = AVG_KEY(open) + спред
                                 r.spread_fix)                      -- fallback
                  END
INTO   #daily
FROM   #cal   c
JOIN   #rolls r
       -- ⬇️ было: c.d BETWEEN r.dt_open AND DATEADD(day,-1,r.dt_close)
       ON c.d BETWEEN r.dt_open AND r.dt_close                       -- включаем день закрытия
LEFT  JOIN #key_spot ks ON ks.DT_REP = c.d
LEFT  JOIN #key_open fk ON fk.DT_REP = r.dt_open AND fk.TERM = r.termdays;

Эти две правки делают поведение таким:
	•	Старый вклад считается включительно по день dt_close.
	•	Новый цикл начинается на следующий день (dt_close + 1).
	•	На 2025-08-20 у тебя больше не будет «ранних» ролловеров — прогнозный средневзвешенный совпадёт с фактом из vw_balance_rest_all.

Доп. замечания (чтобы не словить другие расхождения)
	•	Если есть действительно плавающие вклады ФЛ (редко), которые пересчитываются не по ежедневному spot, а, например, раз в месяц с лагом, то блок is_floatrate=1 потребует учёта даты пересмотра и лага индекса — иначе на конкретный день возможна небольшая дельта.
	•	Для нестандартных сроков вне диапазонов #term_rng ты корректно падаешь в fallback (rate_con - key_avg_open), так что на якорную дату это не должно вносить ошибку.
	•	Убедись, что vw_balance_rest_all интерпретирует OUT_RUB на конец дня; при такой трактовке именно «включительный» dt_close (как выше) даёт совпадение.

Если хочешь оставить обе трактовки под переключателем, можно добавить параметр:

DECLARE @roll_same_day bit = 0;  -- 0 = старт с D+1 (как в факте), 1 = в тот же день

-- и в расчёте next dt_open:
DATEADD(day, s.termdays + CASE WHEN @roll_same_day=1 THEN 0 ELSE 1 END, s.dt_open)
-- и диапазон в #daily:
c.d BETWEEN r.dt_open AND DATEADD(day, CASE WHEN @roll_same_day=1 THEN -1 ELSE 0 END, r.dt_close)

Этого достаточно, чтобы «депозиты закрываемые 20 числа» в твоей витрине вели себя так же, как в факте на 2025-08-20, и средневзвешенная ставка совпала.
