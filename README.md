Хорошо, ты хочешь разделить накопительные счета (НС) по prod_id, чтобы, например, отдельно увидеть аналитику по “НС Ультра”, “НС Базовый” и т.д., и сравнить 30.07 и 02.08 в разрезе продуктов.

⸻

✅ Что будет сделано
	•	Разделим НС по prod_id;
	•	Вклады (section_name = 'Срочные') оставим как единый портфель (если нужно — могу тоже разбить);
	•	Построим финальную таблицу, где для каждого prod_id накопительных счетов будут:
	•	объёмы (0730 / 0208 / Δ);
	•	ставки (0730 / 0208 / Δ);
	•	По вкладам — общие цифры без prod_id.

⸻

✅ Обновлённый запрос

WITH target_clients AS (
    SELECT DISTINCT cli_id
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-07-30'
        AND section_name = 'Накопительный счёт'
        AND od_flag = 1
        AND MONTH(dt_open) IN (5, 6)
),

data_0730 AS (
    SELECT *
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-07-30'
        AND od_flag = 1
        AND section_name IN ('Накопительный счёт', 'Срочные')
        AND cli_id IN (SELECT cli_id FROM target_clients)
),

data_0208 AS (
    SELECT *
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-08-02'
        AND od_flag = 1
        AND section_name IN ('Накопительный счёт', 'Срочные')
        AND cli_id IN (SELECT cli_id FROM target_clients)
),

-- агрегаты по НС с разбивкой по prod_id
agg_ns_0730 AS (
    SELECT
        prod_id,
        SUM(OUT_RUB) AS ns_out_rub_0730,
        SUM(OUT_RUB * con_rate) AS ns_weighted_rate_0730,
        SUM(OUT_RUB) AS ns_base_0730
    FROM data_0730
    WHERE section_name = 'Накопительный счёт'
    GROUP BY prod_id
),

agg_ns_0208 AS (
    SELECT
        prod_id,
        SUM(OUT_RUB) AS ns_out_rub_0208,
        SUM(OUT_RUB * con_rate) AS ns_weighted_rate_0208,
        SUM(OUT_RUB) AS ns_base_0208
    FROM data_0208
    WHERE section_name = 'Накопительный счёт'
    GROUP BY prod_id
),

-- общий агрегат по срочным (не разбиваем по prod_id)
agg_dep_0730 AS (
    SELECT
        SUM(OUT_RUB) AS dep_out_rub_0730,
        SUM(OUT_RUB * con_rate) AS dep_weighted_rate_0730,
        SUM(OUT_RUB * termdays) AS dep_weighted_term_0730,
        SUM(OUT_RUB) AS dep_base_0730
    FROM data_0730
    WHERE section_name = 'Срочные'
),

agg_dep_0208 AS (
    SELECT
        SUM(OUT_RUB) AS dep_out_rub_0208,
        SUM(OUT_RUB * con_rate) AS dep_weighted_rate_0208,
        SUM(OUT_RUB * termdays) AS dep_weighted_term_0208,
        SUM(OUT_RUB) AS dep_base_0208
    FROM data_0208
    WHERE section_name = 'Срочные'
)

-- Финальный вывод
SELECT
    COALESCE(a.prod_id, b.prod_id) AS prod_id,

    -- ===== ОБЪЕМЫ =====
    a.ns_out_rub_0730,
    b.ns_out_rub_0208,
    b.ns_out_rub_0208 - a.ns_out_rub_0730 AS delta_ns_out_rub,

    -- ===== СТАВКИ =====
    CASE WHEN a.ns_base_0730 > 0 THEN a.ns_weighted_rate_0730 / a.ns_base_0730 ELSE NULL END AS ns_avg_rate_0730,
    CASE WHEN b.ns_base_0208 > 0 THEN b.ns_weighted_rate_0208 / b.ns_base_0208 ELSE NULL END AS ns_avg_rate_0208,
    CASE 
        WHEN a.ns_base_0730 > 0 AND b.ns_base_0208 > 0 THEN 
            (b.ns_weighted_rate_0208 / b.ns_base_0208) - (a.ns_weighted_rate_0730 / a.ns_base_0730)
        ELSE NULL
    END AS delta_ns_rate,

    -- ===== ВКЛАДЫ (общие) =====
    dep_0730.dep_out_rub_0730,
    dep_0208.dep_out_rub_0208,
    dep_0208.dep_out_rub_0208 - dep_0730.dep_out_rub_0730 AS delta_dep_out_rub,

    CASE WHEN dep_0730.dep_base_0730 > 0 THEN dep_0730.dep_weighted_rate_0730 / dep_0730.dep_base_0730 ELSE NULL END AS dep_avg_rate_0730,
    CASE WHEN dep_0208.dep_base_0208 > 0 THEN dep_0208.dep_weighted_rate_0208 / dep_0208.dep_base_0208 ELSE NULL END AS dep_avg_rate_0208,
    CASE 
        WHEN dep_0730.dep_base_0730 > 0 AND dep_0208.dep_base_0208 > 0 THEN
            (dep_0208.dep_weighted_rate_0208 / dep_0208.dep_base_0208) - (dep_0730.dep_weighted_rate_0730 / dep_0730.dep_base_0730)
        ELSE NULL
    END AS delta_dep_rate,

    CASE WHEN dep_0730.dep_base_0730 > 0 THEN dep_0730.dep_weighted_term_0730 / dep_0730.dep_base_0730 ELSE NULL END AS dep_avg_term_0730,
    CASE WHEN dep_0208.dep_base_0208 > 0 THEN dep_0208.dep_weighted_term_0208 / dep_0208.dep_base_0208 ELSE NULL END AS dep_avg_term_0208,
    CASE 
        WHEN dep_0730.dep_base_0730 > 0 AND dep_0208.dep_base_0208 > 0 THEN
            (dep_0208.dep_weighted_term_0208 / dep_0208.dep_base_0208) - (dep_0730.dep_weighted_term_0730 / dep_0730.dep_base_0730)
        ELSE NULL
    END AS delta_dep_term

FROM agg_ns_0730 a
FULL OUTER JOIN agg_ns_0208 b ON a.prod_id = b.prod_id
CROSS JOIN agg_dep_0730 dep_0730
CROSS JOIN agg_dep_0208 dep_0208


⸻

📌 Что изменилось
	•	По НС (section_name = 'Накопительный счёт') идёт группировка по prod_id;
	•	По вкладам ('Срочные') остался один общий агрегат;
	•	Всё в одном запросе: объёмы, ставки, срочность и дельты.

⸻

Если тебе нужно сделать отдельные строки по prod_id также и для вкладов — скажи, добавлю.
