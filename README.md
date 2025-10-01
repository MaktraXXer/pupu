Ок, делаем максимально просто и «в лоб», без кумулятивов и хитростей — только календарь, кросс-джойн с сегментом и фильтр по dt_close:
	•	Амортизация: на каждую дату берём все вклады, у которых dt_close_d > эта_дата. Если уже всё вышло — out_rub = 0, ставки NULL.
	•	Выходы: на каждую дату берём ровно те, у кого dt_close_d = эта_дата. Если в день нет выходов — NULL в значениях.

Я оставил твой фильтр по «Эскроу ФЛ / Розничный бизнес», поменял только календарь (чтобы не создавать лишний день) и расчёт.

⸻

Скрипт 1 — Амортизация (включая 2025-08-31 и финальный нулевой день)

DECLARE @dt_rep date = '2025-08-31';

IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
IF OBJECT_ID('tempdb..#cal')  IS NOT NULL DROP TABLE #cal;
IF OBJECT_ID('tempdb..#seg')  IS NOT NULL DROP TABLE #seg;

-- 1) Снимок портфеля на дату отчёта (Эскроу ФЛ, Розничный бизнес)
SELECT
    t.TSegmentname,
    CAST(t.dt_close AS date) AS dt_close_d,
    t.out_rub,
    t.rate_con,
    t.rate_trf
INTO #base
FROM alm.[ALM].[vw_balance_rest_all] t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep
  AND t.AP           = N'Пассив'
  AND t.BLOCK_NAME   = N'Эскроу'
  AND t.section_name = N'Эскроу без ПФ'
  AND t.cur          = '810'
  AND t.acc_role     = N'LIAB'
  AND t.out_rub      IS NOT NULL
  AND t.tprod_name   = N'Эскроу ФЛ'
  AND t.TSegmentname = N'Розничный бизнес'
  AND t.dt_close     > t.dt_rep;      -- закрытия после даты отчёта

-- 2) Диапазон дат: от @dt_rep до макс. dt_close (включительно)
DECLARE @d_end date;
SELECT @d_end = ISNULL(MAX(dt_close_d), @dt_rep) FROM #base;

;WITH cal AS (
    SELECT @dt_rep AS d
    UNION ALL
    SELECT DATEADD(DAY, 1, d) FROM cal WHERE d < @d_end  -- ВАЖНО: <, чтобы включить @d_end и не перелезть дальше
)
SELECT d INTO #cal FROM cal
OPTION (MAXRECURSION 0);

-- 3) Сегменты (здесь один, но оставим общий вид)
SELECT DISTINCT TSegmentname INTO #seg FROM #base;

-- 4) Амортизация: живые вклады = те, у кого dt_close_d > дата
SELECT
    c.d                                      AS [date],
    s.TSegmentname                           AS tsegmentname,
    /* Если уже всё вышло в этот день — сумма будет NULL, приводим к 0 */
    COALESCE(SUM(CASE WHEN b.dt_close_d > c.d THEN b.out_rub END), 0) AS out_rub,
    /* Средневзвешенная TRF по живым; если нет живых — NULL */
    CAST(
        SUM(CASE WHEN b.dt_close_d > c.d AND b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END)
        / NULLIF(SUM(CASE WHEN b.dt_close_d > c.d AND b.rate_trf IS NOT NULL THEN b.out_rub END), 0)
        AS DECIMAL(12,6)
    ) AS rate_trf_srvz,
    /* Средневзвешенная CON по живым; если нет живых — NULL */
    CAST(
        SUM(CASE WHEN b.dt_close_d > c.d AND b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END)
        / NULLIF(SUM(CASE WHEN b.dt_close_d > c.d AND b.rate_con IS NOT NULL THEN b.out_rub END), 0)
        AS DECIMAL(12,6)
    ) AS rate_con_srvz
FROM #cal c
CROSS JOIN #seg s
LEFT JOIN #base b
       ON b.TSegmentname = s.TSegmentname
GROUP BY c.d, s.TSegmentname
ORDER BY c.d, s.TSegmentname;

	•	На последнюю дату (@d_end) строка будет: out_rub = 0, ставки NULL — потому что условие b.dt_close_d > c.d уже нигде не выполняется.

⸻

Скрипт 2 — Выходы (на каждую дату; если нет выходов — NULL)

DECLARE @dt_rep date = '2025-08-31';

IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
IF OBJECT_ID('tempdb..#cal')  IS NOT NULL DROP TABLE #cal;
IF OBJECT_ID('tempdb..#seg')  IS NOT NULL DROP TABLE #seg;
IF OBJECT_ID('tempdb..#clos') IS NOT NULL DROP TABLE #clos;

-- 1) Снимок (тот же фильтр, что выше)
SELECT
    t.TSegmentname,
    CAST(t.dt_close AS date) AS dt_close_d,
    t.out_rub,
    t.rate_con,
    t.rate_trf
INTO #base
FROM alm.[ALM].[vw_balance_rest_all] t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep
  AND t.AP           = N'Пассив'
  AND t.BLOCK_NAME   = N'Эскроу'
  AND t.section_name = N'Эскроу без ПФ'
  AND t.cur          = '810'
  AND t.acc_role     = N'LIAB'
  AND t.out_rub      IS NOT NULL
  AND t.tprod_name   = N'Эскроу ФЛ'
  AND t.TSegmentname = N'Розничный бизнес'
  AND t.dt_close     > t.dt_rep;

-- 2) Диапазон и календарь
DECLARE @d_end date;
SELECT @d_end = ISNULL(MAX(dt_close_d), @dt_rep) FROM #base;

;WITH cal AS (
    SELECT @dt_rep AS d
    UNION ALL
    SELECT DATEADD(DAY, 1, d) FROM cal WHERE d < @d_end
)
SELECT d INTO #cal FROM cal
OPTION (MAXRECURSION 0);

-- 3) Сегменты
SELECT DISTINCT TSegmentname INTO #seg FROM #base;

-- 4) Агрегат «что закрывается в этот день»
SELECT
    b.TSegmentname,
    b.dt_close_d AS d,
    SUM(b.out_rub)                                                    AS out_rub,
    SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub * b.rate_trf END) AS trf_num,
    SUM(CASE WHEN b.rate_trf IS NOT NULL THEN b.out_rub END)              AS trf_den,
    SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END) AS con_num,
    SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END)              AS con_den
INTO #clos
FROM #base b
GROUP BY b.TSegmentname, b.dt_close_d;

-- 5) Выходы: если в дату нет закрытий — значения NULL (как просили)
SELECT
    c.d                               AS [date],
    s.TSegmentname                    AS tsegmentname,
    cl.out_rub                        AS out_rub,
    CAST(cl.trf_num / NULLIF(cl.trf_den, 0) AS DECIMAL(12,6)) AS rate_trf_srvz,
    CAST(cl.con_num / NULLIF(cl.con_den, 0) AS DECIMAL(12,6)) AS rate_con_srvz
FROM #cal c
CROSS JOIN #seg s
LEFT JOIN #clos cl
       ON cl.TSegmentname = s.TSegmentname
      AND cl.d            = c.d
ORDER BY c.d, s.TSegmentname;

Замечания
	•	В твоём примере было ASDECIMAL(12,6) — там нужен пробел: AS DECIMAL(12,6), я поправил.
	•	Если #base вдруг пустая, #seg тоже будет пустой → итог пуст. В таком случае можно принудительно задать сегмент (если нужен) или не фильтровать TSegmentname в выборке снимка.
	•	Если понадобится «Итого по сегментам» (в этом кейсе сегмент один), можно добавить ещё один GROUP BY по дате без сегмента отдельным запросом.

Хочешь объединю амортизацию и выходы в один вывод с меткой metric = 'amortization'/'exits'?
