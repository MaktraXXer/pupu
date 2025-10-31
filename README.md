Ок, вот ровно куда и что заменить — без рассуждений, только патчи.

#1. Заменить блок формирования #firstday_assigned
Найди в твоём скрипте комментарий:

/* ── 6) 1-е числа — ПОЛНЫЙ перелив Σ клиента на max ставку ─── */
IF OBJECT_ID('tempdb..#firstday_assigned') IS NOT NULL DROP TABLE #firstday_assigned;
;WITH ranked AS (
    ...
)
SELECT
    con_id        = MAX(CASE WHEN rn=1 THEN con_id END),
    r.cli_id,
    TSEGMENTNAME  = MAX(CASE WHEN rn=1 THEN TSEGMENTNAME END),
    out_rub       = cs.out_rub_sum,
    r.dt_rep,
    rate_con      = MAX(CASE WHEN rn=1 THEN rate_con END)
INTO   #firstday_assigned
FROM   ranked r
JOIN   #cli_sum cs ON cs.cli_id = r.cli_id
GROUP  BY r.cli_id, r.dt_rep, cs.out_rub_sum;

❗️ВЕСЬ этот блок замени на:

/* ── 6) 1-е числа — ПОЛНЫЙ перелив Σ клиента на max ставку (dedup строго 1 строка/клиент/день) ─ */
IF OBJECT_ID('tempdb..#firstday_assigned') IS NOT NULL DROP TABLE #firstday_assigned;
;WITH ranked AS (
    SELECT
        c.*,
        ROW_NUMBER() OVER (
            PARTITION BY c.cli_id, c.dt_rep
            ORDER BY c.rate_con DESC, c.TSEGMENTNAME, c.con_id
        ) AS rn
    FROM #day1_candidates c
)
SELECT
    r.con_id,
    r.cli_id,
    r.TSEGMENTNAME,
    out_rub  = cs.out_rub_sum,   -- весь объём клиента (Σ на Anchor)
    r.dt_rep,
    r.rate_con
INTO #firstday_assigned
FROM ranked r
JOIN #cli_sum cs
  ON cs.cli_id = r.cli_id
WHERE r.rn = 1;

/* контроль: должно быть пусто */
-- SELECT cli_id, dt_rep, COUNT(*) cnt
-- FROM #firstday_assigned
-- GROUP BY cli_id, dt_rep
-- HAVING COUNT(*) > 1;

#2. Заменить блок формирования #carry_forward
Найди в твоём скрипте комментарий:

/* ── 6.5) Carry-forward ПО ФАКТУ ВЫБОРА 1-го ЧИСЛА:
         берём ТОЧНО rate_con с 1-го, тащим до promo_end ─────── */
IF OBJECT_ID('tempdb..#carry_forward') IS NOT NULL DROP TABLE #carry_forward;
;WITH pick AS ( ... ), bind AS ( ... ), days AS ( ... )
SELECT
    cf.con_id,
    cf.cli_id,
    cf.TSEGMENTNAME,
    out_rub = cs.out_rub_sum,
    cf.dt_rep,
    rate_con = cf.rate_on_first
INTO   #carry_forward
FROM   days cf
JOIN   #cli_sum cs ON cs.cli_id  = cf.cli_id;

❗️ВЕСЬ этот блок замени на:

/* ── 6.5) Carry-forward: фиксируем РОВНО ту ставку, что выбрана на 1-е; 1 строка/клиент/день ─ */
IF OBJECT_ID('tempdb..#carry_forward') IS NOT NULL DROP TABLE #carry_forward;
;WITH bind AS (
    SELECT f.cli_id, f.con_id, f.TSEGMENTNAME, f.dt_rep AS first_day, f.rate_con AS rate_on_first,
           c.cycle_no, c.win_start, c.promo_end, c.base_day
    FROM   #firstday_assigned f
    JOIN   #cycles c
      ON   c.con_id = f.con_id
     AND   f.first_day BETWEEN c.win_start AND c.base_day
),
days AS (
    SELECT b.cli_id, b.con_id, b.TSEGMENTNAME, b.rate_on_first,
           d.d AS dt_rep
    FROM   bind b
    JOIN   #cal d
      ON   d.d BETWEEN DATEADD(day,1,b.first_day) AND b.promo_end   -- строго со 2-го числа
)
SELECT
    con_id       = MIN(con_id),           -- ярлык
    cli_id,
    TSEGMENTNAME = MIN(TSEGMENTNAME),     -- ярлык
    out_rub      = MAX(cs.out_rub_sum),   -- ровно Σ клиента
    dt_rep,
    rate_con     = MAX(rate_on_first)     -- ровно ставка 1-го числа
INTO #carry_forward
FROM days d
JOIN #cli_sum cs
  ON cs.cli_id = d.cli_id
GROUP BY cli_id, dt_rep;

/* контроль: должно быть пусто */
-- SELECT cli_id, dt_rep, COUNT(*) cnt
-- FROM #carry_forward
-- GROUP BY cli_id, dt_rep
-- HAVING COUNT(*) > 1;

#3. Финальная сборка WORK.Forecast_NS_Promo — оставь как у тебя сейчас
У тебя уже правильно стоят три NOT EXISTS-фильтра на #base_dates, #firstday_assigned, #carry_forward и добавляются три слоя UNION ALL. Их не меняем.

⸻

Этого достаточно, чтобы:
	•	на 1-е число был ровно один победитель/клиент/день и полный перелив Σ объёма клиента на эту ставку;
	•	со 2-го числа до promo_end клиент сидел на той же ставке, без пересчётов и без размножения строк;
	•	суммарный объём по всем клиентам в любой день (кроме твоих спец-дней с бизнес-логикой) равнялся @PromoTotal, а не 1.3×/1.5× от него.

Если после вставки увидишь отклонение out_rub_total - @PromoTotal ≠ 0 — быстро проверь (должно быть пусто):

-- дубликаты в 1-х числах
SELECT cli_id, dt_rep, COUNT(*) cnt
FROM #firstday_assigned
GROUP BY cli_id, dt_rep
HAVING COUNT(*) > 1;

-- дубликаты в carry-forward
SELECT cli_id, dt_rep, COUNT(*) cnt
FROM #carry_forward
GROUP BY cli_id, dt_rep
HAVING COUNT(*) > 1;

Скопируй и замени только два указанных блока (#1 и #2). Всё остальное трогать не нужно.
