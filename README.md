Отлично, значит нужно:

📊 Сравнить портфельные метрики на три даты:
	•	2025-06-30 — конец июня,
	•	2025-07-30 — основная дата отбора клиентов,
	•	2025-08-02 — проверка, как переложились.

📌 Все показатели считаются только по клиентам, у которых НС был открыт в мае или июне по срезу на 2025-07-30.

⸻

✅ Унифицированный скрипт для трёх дат:

-- 1. Отбор клиентов с НС, открытым в мае-июне (по балансу на 30.07)
WITH target_clients AS (
    SELECT DISTINCT cli_id
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-07-30'
        AND section_name = 'Накопительный счёт'
        AND od_flag = 1
        AND MONTH(dt_open) IN (5, 6)
),

-- 2. Универсальная таблица со всеми нужными данными за 3 даты
all_data AS (
    SELECT 
        dt_rep,
        cli_id,
        section_name,
        OUT_RUB,
        con_rate,
        termdays
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        dt_rep IN ('2025-06-30', '2025-07-30', '2025-08-02')
        AND od_flag = 1
        AND section_name IN ('Накопительный счёт', 'Срочные')
        AND cli_id IN (SELECT cli_id FROM target_clients)
)

-- 3. Агрегация по каждой дате
SELECT 
    dt_rep,

    -- Объёмы
    SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) AS ns_out_rub,
    SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) AS dep_out_rub,

    -- Ставки
    CASE 
        WHEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) > 0
        THEN SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB * con_rate ELSE 0 END) /
             NULLIF(SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END), 0)
    END AS ns_avg_rate,

    CASE 
        WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
        THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * con_rate ELSE 0 END) /
             NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
    END AS dep_avg_rate,

    -- Срочность
    CASE 
        WHEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) > 0
        THEN SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * termdays ELSE 0 END) /
             NULLIF(SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END), 0)
    END AS dep_avg_termdays

FROM all_data
GROUP BY dt_rep
ORDER BY dt_rep


⸻

📋 Результат:

Таблица из 3 строк:

dt_rep	ns_out_rub	ns_avg_rate	dep_out_rub	dep_avg_rate	dep_avg_termdays
2025-06-30	…	…	…	…	…
2025-07-30	…	…	…	…	…
2025-08-02	…	…	…	…	…


⸻

💡 Хочешь дельты (изменения) между датами?

Могу сразу преобразовать результат в форму:

Метрика	30.06	30.07	02.08	Δ июль	Δ август
Объем НС					
Ставка НС					
Объем вкладов					
Ставка вкладов					
Срочность вкладов					

Готов собрать — просто скажи.
