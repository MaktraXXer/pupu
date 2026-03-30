Да, если ты не закрывал ту же самую сессию / то же самое окно запроса и #bal, #clients_scope еще живы, то из них можно сразу вытащить нужные детальные выгрузки.

Важно:
#temp таблицы существуют только в рамках текущей сессии.
Если окно закрыто, переподключился, или запускаешь в другом окне — их уже нет.

Ниже сначала покажу, как ответить на твои вопросы из уже созданных temp-таблиц, а потом дам отдельный цельный скрипт, если хочешь сделать это заново и нормально.

⸻

1. Можно ли из уже созданных #bal и #clients_scope получить ответ?

Да.

Твой текущий скрипт уже содержит почти все нужные данные.
Просто итоговый SELECT у тебя агрегированный, а деталь можно достать отдельными запросами.

⸻

2. Проверить, живы ли temp-таблицы

IF OBJECT_ID('tempdb..#bal') IS NOT NULL
    SELECT ' #bal exists' AS msg;
ELSE
    SELECT '#bal not found' AS msg;

IF OBJECT_ID('tempdb..#clients_scope') IS NOT NULL
    SELECT '#clients_scope exists' AS msg;
ELSE
    SELECT '#clients_scope not found' AS msg;


⸻

3. Детальная выгрузка: все вклады, которые выходили в заданный период

Это список клиентов и договоров, которые на понедельничном срезе были срочными вкладами и должны были закрыться во вторник–факт.дату.

DECLARE @MondayStart date = '2026-03-23';
DECLARE @FactEndDate date = '2026-03-27';

DECLARE @WeekFrom date = DATEADD(day, 1, @MondayStart);
DECLARE @WeekTo   date = @FactEndDate;

SELECT
      b.cli_id
    , b.con_id
    , b.dt_rep
    , b.dt_open
    , b.dt_close_plan
    , b.section_name
    , b.out_rub
    , b.rate_con
    , b.is_floatrate
    , b.PROD_NAME_res
    , b.TSEGMENTNAME
FROM #bal b
WHERE b.dt_rep = @MondayStart
  AND b.section_name = N'Срочные'
  AND b.dt_close_plan >= @WeekFrom
  AND b.dt_close_plan <= @WeekTo
ORDER BY b.cli_id, b.dt_close_plan, b.con_id;


⸻

4. Детальная выгрузка: кто из этих клиентов открыл новые вклады за период

Это уже только по клиентам из #clients_scope.

DECLARE @MondayStart date = '2026-03-23';
DECLARE @FactEndDate date = '2026-03-27';

DECLARE @WeekFrom date = DATEADD(day, 1, @MondayStart);
DECLARE @WeekTo   date = @FactEndDate;

SELECT
      b.cli_id
    , b.con_id
    , b.dt_rep
    , b.dt_open
    , b.dt_close_plan
    , b.section_name
    , b.out_rub
    , b.rate_con
    , b.is_floatrate
    , b.PROD_NAME_res
    , b.TSEGMENTNAME
FROM #bal b
INNER JOIN #clients_scope c
    ON b.cli_id = c.cli_id
WHERE b.dt_rep = @FactEndDate
  AND b.section_name = N'Срочные'
  AND b.dt_open >= @WeekFrom
  AND b.dt_open <= @WeekTo
ORDER BY b.cli_id, b.dt_open, b.con_id;


⸻

5. НС на начало и конец по этим клиентам

Это по клиентам, у которых выходили вклады.

DECLARE @MondayStart date = '2026-03-23';
DECLARE @FactEndDate date = '2026-03-27';

SELECT
      b.cli_id
    , SUM(CASE WHEN b.dt_rep = @MondayStart THEN b.out_rub ELSE 0 END) AS ns_start_rub
    , SUM(CASE WHEN b.dt_rep = @FactEndDate THEN b.out_rub ELSE 0 END) AS ns_end_rub
    , SUM(CASE WHEN b.dt_rep = @FactEndDate THEN b.out_rub ELSE 0 END)
      - SUM(CASE WHEN b.dt_rep = @MondayStart THEN b.out_rub ELSE 0 END) AS ns_delta_rub
FROM #bal b
INNER JOIN #clients_scope c
    ON b.cli_id = c.cli_id
WHERE b.section_name = N'Накопительный счёт'
GROUP BY b.cli_id
ORDER BY b.cli_id;


⸻

6. Самая полезная сводная деталка: один ряд на клиента, с флагами и суммами

Вот это, скорее всего, тебе и нужно.
Здесь на одного клиента будет:
	•	сколько у него выходило вкладов,
	•	на какую сумму,
	•	сколько открыл новых,
	•	на какую сумму,
	•	НС на начало,
	•	НС на конец.

DECLARE @MondayStart date = '2026-03-23';
DECLARE @FactEndDate date = '2026-03-27';

DECLARE @WeekFrom date = DATEADD(day, 1, @MondayStart);
DECLARE @WeekTo   date = @FactEndDate;

;WITH exit_agg AS
(
    SELECT
          b.cli_id
        , COUNT(DISTINCT b.con_id) AS cnt_con_exit
        , SUM(b.out_rub) AS vol_exit_rub
        , CAST(
            SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END)
            / NULLIF(SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_exit
    FROM #bal b
    WHERE b.dt_rep = @MondayStart
      AND b.section_name = N'Срочные'
      AND b.dt_close_plan >= @WeekFrom
      AND b.dt_close_plan <= @WeekTo
    GROUP BY b.cli_id
),
open_agg AS
(
    SELECT
          b.cli_id
        , COUNT(DISTINCT b.con_id) AS cnt_con_open
        , SUM(b.out_rub) AS vol_open_rub
        , CAST(
            SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub * b.rate_con END)
            / NULLIF(SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_open
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.dt_rep = @FactEndDate
      AND b.section_name = N'Срочные'
      AND b.dt_open >= @WeekFrom
      AND b.dt_open <= @WeekTo
    GROUP BY b.cli_id
),
ns_agg AS
(
    SELECT
          b.cli_id
        , SUM(CASE WHEN b.dt_rep = @MondayStart THEN b.out_rub ELSE 0 END) AS ns_start_rub
        , SUM(CASE WHEN b.dt_rep = @FactEndDate THEN b.out_rub ELSE 0 END) AS ns_end_rub
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.section_name = N'Накопительный счёт'
    GROUP BY b.cli_id
)
SELECT
      c.cli_id
    , ISNULL(e.cnt_con_exit, 0) AS cnt_con_exit
    , ISNULL(e.vol_exit_rub, 0) AS vol_exit_rub
    , e.wavg_rate_exit
    , ISNULL(o.cnt_con_open, 0) AS cnt_con_open
    , ISNULL(o.vol_open_rub, 0) AS vol_open_rub
    , o.wavg_rate_open
    , ISNULL(n.ns_start_rub, 0) AS ns_start_rub
    , ISNULL(n.ns_end_rub, 0) AS ns_end_rub
    , ISNULL(n.ns_end_rub, 0) - ISNULL(n.ns_start_rub, 0) AS ns_delta_rub
    , CASE WHEN ISNULL(o.cnt_con_open, 0) > 0 THEN 1 ELSE 0 END AS opened_new_dep_flag
FROM #clients_scope c
LEFT JOIN exit_agg e
    ON c.cli_id = e.cli_id
LEFT JOIN open_agg o
    ON c.cli_id = o.cli_id
LEFT JOIN ns_agg n
    ON c.cli_id = n.cli_id
ORDER BY vol_exit_rub DESC, c.cli_id;


⸻

7. Если нужна деталка именно “по каждому con_id клиента + НС на начало/конец рядом”

Тогда можно сделать так:
одной таблицей показать каждое исходящее con_id, а рядом клиентские НС и агрегат открытия.

DECLARE @MondayStart date = '2026-03-23';
DECLARE @FactEndDate date = '2026-03-27';

DECLARE @WeekFrom date = DATEADD(day, 1, @MondayStart);
DECLARE @WeekTo   date = @FactEndDate;

;WITH exit_detail AS
(
    SELECT
          b.cli_id
        , b.con_id AS exit_con_id
        , b.dt_open AS exit_dt_open
        , b.dt_close_plan AS exit_dt_close_plan
        , b.out_rub AS exit_out_rub
        , b.rate_con AS exit_rate_con
        , b.PROD_NAME_res AS exit_prod_name
    FROM #bal b
    WHERE b.dt_rep = @MondayStart
      AND b.section_name = N'Срочные'
      AND b.dt_close_plan >= @WeekFrom
      AND b.dt_close_plan <= @WeekTo
),
open_agg AS
(
    SELECT
          b.cli_id
        , COUNT(DISTINCT b.con_id) AS cnt_con_open
        , SUM(b.out_rub) AS vol_open_rub
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.dt_rep = @FactEndDate
      AND b.section_name = N'Срочные'
      AND b.dt_open >= @WeekFrom
      AND b.dt_open <= @WeekTo
    GROUP BY b.cli_id
),
ns_agg AS
(
    SELECT
          b.cli_id
        , SUM(CASE WHEN b.dt_rep = @MondayStart THEN b.out_rub ELSE 0 END) AS ns_start_rub
        , SUM(CASE WHEN b.dt_rep = @FactEndDate THEN b.out_rub ELSE 0 END) AS ns_end_rub
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.section_name = N'Накопительный счёт'
    GROUP BY b.cli_id
)
SELECT
      e.cli_id
    , e.exit_con_id
    , e.exit_dt_open
    , e.exit_dt_close_plan
    , e.exit_out_rub
    , e.exit_rate_con
    , e.exit_prod_name
    , ISNULL(o.cnt_con_open, 0) AS cnt_con_open_by_cli
    , ISNULL(o.vol_open_rub, 0) AS vol_open_rub_by_cli
    , ISNULL(n.ns_start_rub, 0) AS ns_start_rub
    , ISNULL(n.ns_end_rub, 0) AS ns_end_rub
    , ISNULL(n.ns_end_rub, 0) - ISNULL(n.ns_start_rub, 0) AS ns_delta_rub
FROM exit_detail e
LEFT JOIN open_agg o
    ON e.cli_id = o.cli_id
LEFT JOIN ns_agg n
    ON e.cli_id = n.cli_id
ORDER BY e.cli_id, e.exit_dt_close_plan, e.exit_con_id;


⸻

8. Можно ли из твоего уже выполненного скрипта понять ответ “на мои вопросы”?

Да, но частично:

Твой скрипт в текущем виде уже дает:
	•	сколько было выходящих вкладов,
	•	сколько было открытых вкладов,
	•	общий объем,
	•	НС на начало и конец,
	•	но только в агрегате по всей группе.

Чтобы ответить:
	•	какие именно клиенты,
	•	какие именно con_id,
	•	кто из них открыл вклад,
	•	какие НС у конкретного клиента на начало и конец,

нужно делать дополнительные запросы к #bal и #clients_scope — те, что я выше написал.

⸻

9. Если хочешь отдельный цельный скрипт — вот нормальный вариант

Он сразу формирует 3 детальные выгрузки:
	1.	#exit_detail — выходящие вклады
	2.	#open_detail — открытые новые вклады этих клиентов
	3.	#client_summary — клиентская сводка с НС на начало/конец

USE [ALM];
SET NOCOUNT ON;

/* ============================================
   ПАРАМЕТРЫ
   ============================================ */
DECLARE @MondayStart date = '2026-03-23';
DECLARE @FactEndDate date = '2026-03-27';

/* ============================================
   ПРОВЕРКИ
   ============================================ */
IF (DATEDIFF(day, '19000101', @MondayStart) % 7) <> 0
BEGIN
    RAISERROR(N'@MondayStart должен быть понедельником.', 16, 1);
    RETURN;
END;

DECLARE @ReliableDate date = DATEADD(day, -2, CAST(GETDATE() AS date));

IF @FactEndDate > @ReliableDate
BEGIN
    RAISERROR(N'@FactEndDate не может быть больше GETDATE()-2.', 16, 1);
    RETURN;
END;

IF @FactEndDate <= @MondayStart
BEGIN
    RAISERROR(N'@FactEndDate должен быть больше @MondayStart.', 16, 1);
    RETURN;
END;

DECLARE @WeekFrom date = DATEADD(day, 1, @MondayStart);
DECLARE @WeekTo   date = @FactEndDate;

IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;
IF OBJECT_ID('tempdb..#clients_scope') IS NOT NULL DROP TABLE #clients_scope;
IF OBJECT_ID('tempdb..#exit_detail') IS NOT NULL DROP TABLE #exit_detail;
IF OBJECT_ID('tempdb..#open_detail') IS NOT NULL DROP TABLE #open_detail;
IF OBJECT_ID('tempdb..#client_summary') IS NOT NULL DROP TABLE #client_summary;

/* ============================================
   БАЛАНС
   ============================================ */
SELECT
      CAST(t.dt_rep AS date) AS dt_rep
    , CAST(t.cli_id AS bigint) AS cli_id
    , CAST(t.con_id AS bigint) AS con_id
    , CAST(t.dt_open AS date) AS dt_open
    , CAST(t.dt_close_plan AS date) AS dt_close_plan
    , CAST(t.section_name AS nvarchar(50)) AS section_name
    , CAST(t.out_rub AS decimal(38,6)) AS out_rub
    , CAST(t.rate_con AS decimal(18,6)) AS rate_con
    , CAST(ISNULL(t.is_floatrate,0) AS bit) AS is_floatrate
    , CAST(t.PROD_NAME_res AS nvarchar(255)) AS PROD_NAME_res
    , CAST(t.TSEGMENTNAME AS nvarchar(255)) AS TSEGMENTNAME
INTO #bal
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep IN (@MondayStart, @FactEndDate)
  AND t.section_name IN (N'Срочные', N'Накопительный счёт')
  AND t.block_name = N'Привлечение ФЛ'
  AND t.acc_role   = N'LIAB'
  AND t.cur        = '810'
  AND t.od_flag    = 1
  AND t.out_rub IS NOT NULL
  AND t.out_rub >= 0;

CREATE CLUSTERED INDEX CIX_#bal
    ON #bal (dt_rep, section_name, cli_id, con_id);

/* ============================================
   КЛИЕНТЫ В СКОУПЕ
   ============================================ */
SELECT DISTINCT
    b.cli_id
INTO #clients_scope
FROM #bal b
WHERE b.dt_rep = @MondayStart
  AND b.section_name = N'Срочные'
  AND b.dt_close_plan >= @WeekFrom
  AND b.dt_close_plan <= @WeekTo;

CREATE UNIQUE CLUSTERED INDEX CIX_#clients_scope
    ON #clients_scope(cli_id);

/* ============================================
   1. ДЕТАЛЬ ВЫХОДЯЩИХ ВКЛАДОВ
   ============================================ */
SELECT
      b.cli_id
    , b.con_id
    , b.dt_open
    , b.dt_close_plan
    , b.out_rub
    , b.rate_con
    , b.is_floatrate
    , b.PROD_NAME_res
    , b.TSEGMENTNAME
INTO #exit_detail
FROM #bal b
WHERE b.dt_rep = @MondayStart
  AND b.section_name = N'Срочные'
  AND b.dt_close_plan >= @WeekFrom
  AND b.dt_close_plan <= @WeekTo;

/* ============================================
   2. ДЕТАЛЬ НОВЫХ ВКЛАДОВ ЭТИХ КЛИЕНТОВ
   ============================================ */
SELECT
      b.cli_id
    , b.con_id
    , b.dt_open
    , b.dt_close_plan
    , b.out_rub
    , b.rate_con
    , b.is_floatrate
    , b.PROD_NAME_res
    , b.TSEGMENTNAME
INTO #open_detail
FROM #bal b
INNER JOIN #clients_scope c
    ON b.cli_id = c.cli_id
WHERE b.dt_rep = @FactEndDate
  AND b.section_name = N'Срочные'
  AND b.dt_open >= @WeekFrom
  AND b.dt_open <= @WeekTo;

/* ============================================
   3. КЛИЕНТСКАЯ СВОДКА
   ============================================ */
;WITH exit_agg AS
(
    SELECT
          cli_id
        , COUNT(DISTINCT con_id) AS cnt_con_exit
        , SUM(out_rub) AS vol_exit_rub
        , CAST(
            SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub * rate_con END)
            / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_exit
    FROM #exit_detail
    GROUP BY cli_id
),
open_agg AS
(
    SELECT
          cli_id
        , COUNT(DISTINCT con_id) AS cnt_con_open
        , SUM(out_rub) AS vol_open_rub
        , CAST(
            SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub * rate_con END)
            / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_open
    FROM #open_detail
    GROUP BY cli_id
),
ns_agg AS
(
    SELECT
          b.cli_id
        , SUM(CASE WHEN b.dt_rep = @MondayStart THEN b.out_rub ELSE 0 END) AS ns_start_rub
        , SUM(CASE WHEN b.dt_rep = @FactEndDate THEN b.out_rub ELSE 0 END) AS ns_end_rub
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.section_name = N'Накопительный счёт'
    GROUP BY b.cli_id
)
SELECT
      c.cli_id
    , ISNULL(e.cnt_con_exit, 0) AS cnt_con_exit
    , ISNULL(e.vol_exit_rub, 0) AS vol_exit_rub
    , e.wavg_rate_exit
    , ISNULL(o.cnt_con_open, 0) AS cnt_con_open
    , ISNULL(o.vol_open_rub, 0) AS vol_open_rub
    , o.wavg_rate_open
    , ISNULL(n.ns_start_rub, 0) AS ns_start_rub
    , ISNULL(n.ns_end_rub, 0) AS ns_end_rub
    , ISNULL(n.ns_end_rub, 0) - ISNULL(n.ns_start_rub, 0) AS ns_delta_rub
    , CASE WHEN ISNULL(o.cnt_con_open, 0) > 0 THEN 1 ELSE 0 END AS opened_new_dep_flag
INTO #client_summary
FROM #clients_scope c
LEFT JOIN exit_agg e
    ON c.cli_id = e.cli_id
LEFT JOIN open_agg o
    ON c.cli_id = o.cli_id
LEFT JOIN ns_agg n
    ON c.cli_id = n.cli_id;

/* ============================================
   ВЫВОДЫ
   ============================================ */

-- 1) Выходящие вклады
SELECT *
FROM #exit_detail
ORDER BY cli_id, dt_close_plan, con_id;

-- 2) Новые вклады тех же клиентов
SELECT *
FROM #open_detail
ORDER BY cli_id, dt_open, con_id;

-- 3) Сводка по клиентам
SELECT *
FROM #client_summary
ORDER BY vol_exit_rub DESC, cli_id;


⸻

10. Что лучше использовать на практике

Если тебе надо быстро проверить уже выполненный запуск — используй запросы к существующим #bal и #clients_scope.

Если хочешь:
	•	сохранить логику,
	•	потом экспортировать,
	•	не зависеть от того, живы temp-таблицы или нет,

то лучше запускать отдельный цельный скрипт из пункта 9.

Если хочешь, я могу сразу сделать еще одну версию: одна итоговая деталка “1 строка = 1 клиент + список exit_con_id/open_con_id не агрегированно” или версию с выгрузкой в постоянные таблицы tempdb..##global / alm.temp_*.
