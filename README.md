Понял — ошибка не в арифметике, а в том, какие строки мы оставляем в портфеле после «перелива»:
	•	я удаляю все FIX-promo строки, кроме агрегированной, но при этом
забываю забрать их объём из исходной таблицы;
в результате объём «раздувается» ровно на сумму promo-FIX и портфель
дублируется.

Ни расчёт спреда, ни ставки тут ни при чём — ломается только этап
union_all.  Достаточно:
	1.	Вычеркнуть promo-FIX из #daily_all, после того, как объединили
три источника ставок;
	2.	Добавить вместо них одну агрегат-строку на клиента/1-е число.

/* ── 9а. “правильная” лента без дублей promo-FIX ────────────── */
DROP TABLE IF EXISTS #daily_mix;
SELECT * INTO #daily_mix
FROM   #daily_all
WHERE  NOT (prod_id = 654                        -- FIX
        AND dt_rep = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1));

-- promo-FIX, склеенные по клиенту (только 1-е число месяца)
WITH first_fix AS (
     SELECT cli_id,
            DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1) AS m1
     FROM   #daily_all
     WHERE  prod_id = 654
       AND  dt_rep  = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
     GROUP  BY cli_id,
              DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1))
SELECT NULL        AS con_id,
       cli_id,
       NULL        AS TSEGMENTNAME,
       SUM(out_rub)                      AS out_rub,
       m1            AS dt_rep,
       MAX(rate_con)                     AS rate_con,
       654           AS prod_id      -- помечаем как FIX
INTO   #promo_glue
FROM   #daily_all d
JOIN   first_fix f
       ON  f.cli_id = d.cli_id
      AND f.m1     = d.dt_rep
WHERE  d.prod_id = 654
GROUP  BY cli_id, m1;

/* ── 9b. окончательная лента для агрегата ───────────────────── */
DROP TABLE IF EXISTS #daily_final;
SELECT * INTO #daily_final
FROM (
      SELECT * FROM #daily_mix
      UNION ALL
      SELECT * FROM #promo_glue ) z;

/* ── 10. итог портфеля ──────────────────────────────────────── */
TRUNCATE TABLE WORK.Forecast_BalanceDaily_NS;

INSERT WORK.Forecast_BalanceDaily_NS
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #daily_final
GROUP  BY dt_rep;

как проверить

-- объём должен быть константой (≈ 127 258 млрд) до 30 августа
SELECT TOP (60) dt_rep,out_rub_total,rate_avg
FROM   WORK.Forecast_BalanceDaily_NS
ORDER  BY dt_rep;

	•	30 августа — базовая 6.5 %
	•	1 сентября — ставка promo (0.173 / 0.1709)
	•	15 сентября — допереоценка только FLOAT
	•	30 сентября — снова базовая
	•	1 окт / ноя / дек — promo 0.153 / 0.1509

Если объём теперь держится ровно и «ступеньки» по ставке
совпадают с логикой — модель заработала.
Не совпало → скинь конкретный датасет (хотя бы пару
cli_id) — разберёмся точечно.
