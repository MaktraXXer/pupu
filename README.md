Абсолютно верно — спасибо за уточнение. Мы не хотим поднимать все исторические dt_open, а хотим использовать только баланс на 2025-07-30, и из него уже выбрать клиентов, у которых накопительный счёт открыт в мае или июне 2025 года (т.е. MONTH(dt_open) in (5,6)).

⸻

✅ Обновлённая корректная логика:
	1.	Берём только dt_rep = '2025-07-30' (срез баланса на дату).
	2.	Внутри него ищем клиентов, у кого:
	•	section_name = 'Накопительный счёт';
	•	MONTH(dt_open) IN (5, 6);
	•	od_flag = 1.
	3.	По этим cli_id строим агрегаты на:
	•	2025-07-30 (входной баланс);
	•	2025-08-02 (проверка изменений).

⸻

✳️ Исправленный SQL-запрос

-- 1. Находим клиентов с НС, открытым в мае-июне по балансу на 30.07
WITH target_clients AS (
    SELECT DISTINCT cli_id
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-07-30'
        AND section_name = 'Накопительный счёт'
        AND od_flag = 1
        AND MONTH(dt_open) IN (5, 6)
),

-- 2. Срез на 30.07
agg_0730 AS (
    SELECT
        br.cli_id,
        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) AS ns_out_rub_0730,
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) AS dep_out_rub_0730,

        -- ставки
        CASE 
            WHEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB * con_rate ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END), 0)
        END AS ns_avg_rate_0730,

        CASE 
            WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * con_rate ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
        END AS dep_avg_rate_0730,

        CASE 
            WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * termdays ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
        END AS dep_avg_termdays_0730

    FROM ALM.vw_balance_rest_all br WITH (NOLOCK)
    JOIN target_clients t ON br.cli_id = t.cli_id
    WHERE br.dt_rep = '2025-07-30'
      AND br.od_flag = 1
      AND br.section_name IN ('Накопительный счёт', 'Срочные')
    GROUP BY br.cli_id
),

-- 3. Срез на 02.08
agg_0208 AS (
    SELECT
        br.cli_id,
        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) AS ns_out_rub_0208,
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) AS dep_out_rub_0208,

        CASE 
            WHEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB * con_rate ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END), 0)
        END AS ns_avg_rate_0208,

        CASE 
            WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * con_rate ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
        END AS dep_avg_rate_0208,

        CASE 
            WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
            THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * termdays ELSE 0 END)
                 / NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
        END AS dep_avg_termdays_0208

    FROM ALM.vw_balance_rest_all br WITH (NOLOCK)
    JOIN target_clients t ON br.cli_id = t.cli_id
    WHERE br.dt_rep = '2025-08-02'
      AND br.od_flag = 1
      AND br.section_name IN ('Накопительный счёт', 'Срочные')
    GROUP BY br.cli_id
)

-- 4. Итоговая сводка
SELECT 
    COALESCE(a.cli_id, j.cli_id) AS cli_id,

    -- 30.07
    j.ns_out_rub_0730,
    j.ns_avg_rate_0730,
    j.dep_out_rub_0730,
    j.dep_avg_rate_0730,
    j.dep_avg_termdays_0730,

    -- 02.08
    a.ns_out_rub_0208,
    a.ns_avg_rate_0208,
    a.dep_out_rub_0208,
    a.dep_avg_rate_0208,
    a.dep_avg_termdays_0208

FROM agg_0730 j
FULL OUTER JOIN agg_0208 a ON a.cli_id = j.cli_id


⸻

📌 Кратко:
	•	Используется только срез на 30.07 для выбора клиентов;
	•	Затем анализируются две даты (30.07 и 02.08);
	•	Результат — полная картина по тем клиентам, у кого НС открыт в мае/июне, и что стало с их остатками и ставками.

Если хочешь — можно добавить дельты (Δ) по суммам и ставкам, или подсчитать общий итог.
