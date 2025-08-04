Спасибо, теперь всё ясно. Задача такая:

⸻

✅ Цель

Шаг 1. Найти клиентов, у которых накопительный счёт (section_name = 'Накопительный счёт') был открыт в мае или июне 2025 года.

Шаг 2. По этим клиентам собрать на определённую дату (например, 2025-08-02) статистику:
	•	объём по НС и срочным вкладам (section_name = 'Срочные');
	•	средневзвешенные con_rate по НС и вкладам;
	•	средневзвешенная termdays по вкладам.

Шаг 3. Повторить тот же расчёт для другой даты (например, 2025-07-30), но по тем же cli_id, чтобы сравнить, как изменилось поведение.

⸻

💡 SQL-запрос

-- 1. Найдём клиентов с НС, открытым в мае или июне
WITH clients_ns_opened_may_jun AS (
    SELECT DISTINCT cli_id
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        section_name = 'Накопительный счёт'
        AND od_flag = 1
        AND dt_open >= '2025-05-01'
        AND dt_open <  '2025-07-01'
),

-- 2. Считаем агрегаты на 02.08 по этим клиентам
agg_0208 AS (
    SELECT
        br.cli_id,
        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) AS ns_out_rub_0208,
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) AS dep_out_rub_0208,

        -- средневзвешенная ставка по НС
        CASE 
            WHEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB * con_rate ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END), 0)
        END AS ns_avg_rate_0208,

        -- средневзвешенная ставка по вкладам
        CASE 
            WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * con_rate ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
        END AS dep_avg_rate_0208,

        -- средневзвешенная срочность по вкладам
        CASE 
            WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * termdays ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
        END AS dep_avg_termdays_0208

    FROM ALM.vw_balance_rest_all br WITH (NOLOCK)
    JOIN clients_ns_opened_may_jun c ON br.cli_id = c.cli_id
    WHERE br.dt_rep = '2025-08-02'
      AND br.od_flag = 1
      AND br.section_name IN ('Накопительный счёт', 'Срочные')
    GROUP BY br.cli_id
),

-- 3. Считаем агрегаты на 30.07 по тем же клиентам
agg_0730 AS (
    SELECT
        br.cli_id,
        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) AS ns_out_rub_0730,
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) AS dep_out_rub_0730,

        -- средневзвешенная ставка по НС
        CASE 
            WHEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB * con_rate ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END), 0)
        END AS ns_avg_rate_0730,

        -- средневзвешенная ставка по вкладам
        CASE 
            WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * con_rate ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
        END AS dep_avg_rate_0730,

        -- средневзвешенная срочность по вкладам
        CASE 
            WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * termdays ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
        END AS dep_avg_termdays_0730

    FROM ALM.vw_balance_rest_all br WITH (NOLOCK)
    JOIN clients_ns_opened_may_jun c ON br.cli_id = c.cli_id
    WHERE br.dt_rep = '2025-07-30'
      AND br.od_flag = 1
      AND br.section_name IN ('Накопительный счёт', 'Срочные')
    GROUP BY br.cli_id
)

-- 4. Объединяем всё
SELECT 
    a.cli_id,

    -- данные на 30.07
    j.ns_out_rub_0730,
    j.ns_avg_rate_0730,
    j.dep_out_rub_0730,
    j.dep_avg_rate_0730,
    j.dep_avg_termdays_0730,

    -- данные на 02.08
    a.ns_out_rub_0208,
    a.ns_avg_rate_0208,
    a.dep_out_rub_0208,
    a.dep_avg_rate_0208,
    a.dep_avg_termdays_0208

FROM agg_0208 a
FULL OUTER JOIN agg_0730 j ON a.cli_id = j.cli_id


⸻

📌 Результат:

Для каждого cli_id:

cli_id	ns_out_rub_0730	ns_avg_rate_0730	…	ns_out_rub_0208	ns_avg_rate_0208	…
	(НС 30.07)	ставка		(НС 02.08)	ставка	


⸻

Если хочешь — можно рассчитать дельты (изменения) между датами или визуализировать динамику.

Готов сделать это следующим шагом.
