Ок—ниже весь «диагностический набор» без временных таблиц. Просто задаёте параметры и запускаете нужные блоки. Все проверки идут напрямую по ALM.ALM.vw_balance_rest_all и WORK.ForecastKey_Cache_Scen. Диапазоны термов я задал через CTE term_rng (VALUES), чтобы не зависеть от #term_rng.

⸻

Параметры (поставьте свои даты/сценарий)

DECLARE @Anchor   date    = '2025-10-17';
DECLARE @scen     tinyint = 1;
DECLARE @win_from date    = DATEADD(day,-3,@Anchor);
DECLARE @win_to   date    = DATEADD(day,+3,@Anchor);


⸻

CTE: номинализация термов (диапазоны → term_nom)

WITH term_rng AS (
  SELECT * FROM (VALUES
    ( 61,  46,  76),
    ( 91,  76, 106),
    (122, 107, 137),
    (181, 166, 196),
    (274, 259, 289),
    (365, 352, 382),
    (548, 535, 565),
    (730, 715, 745),
    (1100,1085,1115)
  ) v(term_nom, term_lo, term_hi)
)
SELECT 1 AS _run;  -- маркер, чтобы CTE подхватился

В следующих запросах этот term_rng подключается повторно как подзапрос (CTE внутри каждого блока), так что можно запускать блоки по отдельности.

⸻

1) Проверка: «остаточный» ли у вас termdays на снимке (на Anchor)

SELECT
  total               = COUNT(*),
  residual_mismatch   = SUM(CASE WHEN b.termdays <> DATEDIFF(day, b.dt_open, b.dt_close) THEN 1 ELSE 0 END),
  examples_to_check   = STRING_AGG(CONVERT(varchar(50), b.con_id), ',') WITHIN GROUP (ORDER BY b.con_id)
FROM ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
WHERE b.dt_rep = @Anchor
  AND b.section_name = N'Срочные'
  AND b.block_name   = N'Привлечение ФЛ'
  AND b.od_flag      = 1
  AND b.cur          = '810';

Если residual_mismatch > 0, значит на снимке termdays не равен «контрактному» DATEDIFF(dt_open,dt_close), и прямой join к ключу по TERM=b.termdays на дату открытия потенциально ломается.

⸻

2) Нет ключа на дату открытия по номиналу (главная причина «ломается на 17-е»)

WITH term_rng AS (
  SELECT * FROM (VALUES
    ( 61,  46,  76),( 91,  76, 106),(122, 107, 137),
    (181, 166, 196),(274, 259, 289),(365, 352, 382),
    (548, 535, 565),(730, 715, 745),(1100,1085,1115)
  ) v(term_nom, term_lo, term_hi)
)
SELECT
  missing_cnt = COUNT(*)
FROM ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
LEFT JOIN term_rng tr
  ON b.termdays BETWEEN tr.term_lo AND tr.term_hi
LEFT JOIN WORK.ForecastKey_Cache_Scen k WITH (NOLOCK)
  ON k.DT_REP = b.dt_open
 AND k.TERM   = tr.term_nom
 AND k.Scenario = @scen
WHERE b.dt_rep = @Anchor
  AND b.section_name = N'Срочные'
  AND b.block_name   = N'Привлечение ФЛ'
  AND b.od_flag      = 1
  AND b.cur          = '810'
  AND tr.term_nom IS NOT NULL
  AND k.TERM IS NULL;

Если missing_cnt > 0 — на часть договоров не нашёлся ключ на дату открытия (по номиналу). Это прямо ведёт к NULL в key_avg_open/fallback и «странной» средневзвешенной ставке.

⸻

3) Какие именно договоры без ключа на дату открытия

WITH term_rng AS (
  SELECT * FROM (VALUES
    ( 61,  46,  76),( 91,  76, 106),(122, 107, 137),
    (181, 166, 196),(274, 259, 289),(365, 352, 382),
    (548, 535, 565),(730, 715, 745),(1100,1085,1115)
  ) v(term_nom, term_lo, term_hi)
)
SELECT TOP (50)
  b.con_id, b.dt_open, b.dt_close, b.termdays, tr.term_nom
FROM ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
LEFT JOIN term_rng tr
  ON b.termdays BETWEEN tr.term_lo AND tr.term_hi
LEFT JOIN WORK.ForecastKey_Cache_Scen k WITH (NOLOCK)
  ON k.DT_REP = b.dt_open
 AND k.TERM   = tr.term_nom
 AND k.Scenario = @scen
WHERE b.dt_rep = @Anchor
  AND b.section_name = N'Срочные'
  AND b.block_name   = N'Привлечение ФЛ'
  AND b.od_flag      = 1
  AND b.cur          = '810'
  AND tr.term_nom IS NOT NULL
  AND k.TERM IS NULL
ORDER BY b.dt_open DESC;


⸻

4) Дубликаты ключа по (DT_REP, TERM, Scenario) вокруг Anchor

SELECT DT_REP, TERM, Scenario, cnt = COUNT(*)
FROM WORK.ForecastKey_Cache_Scen WITH (NOLOCK)
WHERE Scenario = @scen
  AND DT_REP BETWEEN @win_from AND @win_to
GROUP BY DT_REP, TERM, Scenario
HAVING COUNT(*) > 1
ORDER BY DT_REP, TERM;

Есть строки? Тогда при join вы размножаете записи (и веса) — это может заметно смещать среднюю на одной дате, но не на другой.

⸻

5) На 17.10 отсутствуют какие-то «номиналы» ключа

WITH need_terms AS (
  SELECT TERM FROM (VALUES (61),(91),(122),(181),(274),(365),(548),(730),(1100)) v(TERM)
)
SELECT nt.TERM
FROM need_terms nt
LEFT JOIN WORK.ForecastKey_Cache_Scen k WITH (NOLOCK)
  ON k.DT_REP = '2025-10-17'  -- можно заменить на @Anchor
 AND k.TERM   = nt.TERM
 AND k.Scenario = @scen
WHERE k.TERM IS NULL;

Если вывод не пуст — на 17.10 нет ключа для одного/нескольких номиналов.

⸻

6) Покрытие справочником «to-be» по сегментам/конвенциям (без temp-#ref)

Здесь проверяем не сам справочник (он у вас в коде временный), а расклад портфеля по сегментам и конвенциям: что именно должно покрываться «O/R» и AT_THE_END/«monthly». Это быстро покажет, где может быть «дырка» в вашем #ref.

SELECT
  seg  = CASE WHEN b.TSEGMENTNAME = N'Розничный Бизнес' THEN 'R' ELSE 'O' END,
  b.conv,
  cnt  = COUNT(*)
FROM ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
WHERE b.dt_rep = @Anchor
  AND b.section_name = N'Срочные'
  AND b.block_name   = N'Привлечение ФЛ'
  AND b.od_flag      = 1
  AND b.cur          = '810'
GROUP BY CASE WHEN b.TSEGMENTNAME = N'Розничный Бизнес' THEN 'R' ELSE 'O' END, b.conv
ORDER BY 1,2;

Сравните этот расклад с тем, что «разрешает» ваш справочник to_be_O/delta_R/ветки по conv.

⸻

7) На Anchor есть NULL в rate_con (часто ломает только «сегодня»)

SELECT
  total      = COUNT(*),
  null_rates = SUM(CASE WHEN b.rate_con IS NULL THEN 1 ELSE 0 END)
FROM ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
WHERE b.dt_rep = @Anchor
  AND b.section_name = N'Срочные'
  AND b.block_name   = N'Привлечение ФЛ'
  AND b.od_flag      = 1
  AND b.cur          = '810';

Если null_rates > 0 на 17.10, а на 16.10 — 0, у вас ещё не «доприехали» ставки в ETL.

⸻

8) Контракты, не попавшие в ни один диапазон номиналов (границы)

WITH term_rng AS (
  SELECT * FROM (VALUES
    ( 61,  46,  76),( 91,  76, 106),(122, 107, 137),
    (181, 166, 196),(274, 259, 289),(365, 352, 382),
    (548, 535, 565),(730, 715, 745),(1100,1085,1115)
  ) v(term_nom, term_lo, term_hi)
)
SELECT COUNT(*) AS out_of_ranges
FROM ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
LEFT JOIN term_rng tr
  ON b.termdays BETWEEN tr.term_lo AND tr.term_hi
WHERE b.dt_rep = @Anchor
  AND b.section_name = N'Срочные'
  AND b.block_name   = N'Привлечение ФЛ'
  AND b.od_flag      = 1
  AND b.cur          = '810'
  AND tr.term_nom IS NULL;

Если >0 — часть договоров выпала из ваших окон (например, 45 или 1116 дней) и не мэппится к номиналам.

⸻

9) Проверка «правильного» сценария (вдруг запускались с другим @scen)

SELECT Scenario, cnt = COUNT(*)
FROM WORK.ForecastKey_Cache_Scen WITH (NOLOCK)
WHERE DT_REP = @Anchor
GROUP BY Scenario
ORDER BY Scenario;


⸻

10) Ключ на дату нового открытия ролловера (контроль на будущее)

Если вы считаете n≥1 от AVG_KEY_RATE на дату нового открытия, убедитесь, что ключ для этих дат действительно есть.

WITH term_rng AS (
  SELECT * FROM (VALUES
    ( 61,  46,  76),( 91,  76, 106),(122, 107, 137),
    (181, 166, 196),(274, 259, 289),(365, 352, 382),
    (548, 535, 565),(730, 715, 745),(1100,1085,1115)
  ) v(term_nom, term_lo, term_hi)
)
SELECT TOP (50)
  b.con_id,
  next_open = DATEADD(day, tr.term_nom, b.dt_open),
  tr.term_nom,
  has_key   = CASE WHEN EXISTS (
                SELECT 1 FROM WORK.ForecastKey_Cache_Scen k WITH (NOLOCK)
                WHERE k.DT_REP = DATEADD(day, tr.term_nom, b.dt_open)
                  AND k.TERM   = tr.term_nom
                  AND k.Scenario = @scen
              ) THEN 1 ELSE 0 END
FROM ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
JOIN term_rng tr
  ON b.termdays BETWEEN tr.term_lo AND tr.term_hi
WHERE b.dt_rep = @Anchor
  AND b.section_name = N'Срочные'
  AND b.block_name   = N'Привлечение ФЛ'
  AND b.od_flag      = 1
  AND b.cur          = '810'
ORDER BY has_key;


⸻

11) Дубликаты в портфеле на Anchor (удвоенные строки)

SELECT b.con_id, cnt = COUNT(*)
FROM ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
WHERE b.dt_rep = @Anchor
  AND b.section_name = N'Срочные'
  AND b.block_name   = N'Привлечение ФЛ'
  AND b.od_flag      = 1
  AND b.cur          = '810'
GROUP BY b.con_id
HAVING COUNT(*) > 1
ORDER BY cnt DESC;


⸻

12) Быстрая сводка по «рисковым» факторам (в одно окно)

DECLARE @issues TABLE (check_name sysname, value int);

-- NULL rates
INSERT @issues
SELECT 'NULL rate_con', SUM(CASE WHEN rate_con IS NULL THEN 1 ELSE 0 END)
FROM ALM.ALM.vw_balance_rest_all WITH (NOLOCK)
WHERE dt_rep=@Anchor AND section_name=N'Срочные' AND block_name=N'Привлечение ФЛ' AND od_flag=1 AND cur='810';

-- out-of-range termdays
WITH term_rng AS (
  SELECT * FROM (VALUES
    ( 61,  46,  76),( 91,  76, 106),(122, 107, 137),
    (181, 166, 196),(274, 259, 289),(365, 352, 382),
    (548, 535, 565),(730, 715, 745),(1100,1085,1115)
  ) v(term_nom, term_lo, term_hi)
)
INSERT @issues
SELECT 'Out-of-range termdays', COUNT(*)
FROM ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
LEFT JOIN term_rng tr ON b.termdays BETWEEN tr.term_lo AND tr.term_hi
WHERE b.dt_rep=@Anchor AND b.section_name=N'Срочные' AND b.block_name=N'Привлечение ФЛ' AND b.od_flag=1 AND b.cur='810' AND tr.term_nom IS NULL;

-- missing key on dt_open by nominal term
WITH term_rng AS (
  SELECT * FROM (VALUES
    ( 61,  46,  76),( 91,  76, 106),(122, 107, 137),
    (181, 166, 196),(274, 259, 289),(365, 352, 382),
    (548, 535, 565),(730, 715, 745),(1100,1085,1115)
  ) v(term_nom, term_lo, term_hi)
)
INSERT @issues
SELECT 'No key on dt_open', COUNT(*)
FROM ALM.ALM.vw_balance_rest_all b WITH (NOLOCK)
LEFT JOIN term_rng tr ON b.termdays BETWEEN tr.term_lo AND tr.term_hi
LEFT JOIN WORK.ForecastKey_Cache_Scen k WITH (NOLOCK)
       ON k.DT_REP=b.dt_open AND k.TERM=tr.term_nom AND k.Scenario=@scen
WHERE b.dt_rep=@Anchor AND b.section_name=N'Срочные' AND b.block_name=N'Привлечение ФЛ' AND b.od_flag=1 AND b.cur='810'
  AND tr.term_nom IS NOT NULL AND k.TERM IS NULL;

-- key duplicates around anchor
INSERT @issues
SELECT 'Key duplicates (±3d)', SUM(cnt) FROM (
  SELECT COUNT(*) AS cnt
  FROM WORK.ForecastKey_Cache_Scen WITH (NOLOCK)
  WHERE Scenario=@scen AND DT_REP BETWEEN @win_from AND @win_to
  GROUP BY DT_REP, TERM, Scenario
  HAVING COUNT(*)>1
) d;

SELECT * FROM @issues;


⸻

Если по этим проверкам «горит» именно (2)/(3)/(5)/(8) — это и есть объяснение, почему на 17.10 пересчёт ведёт к «неожиданным» средневзвешенным ставкам, а на 16.10 ещё «держалось».
