Вот переработанная версия твоего запроса — корректно и эффективно, без NOT IN, но с исключением клиентов, у которых на 30.07 был НС, открытый в мае или июне.
Это делается через LEFT JOIN ... WHERE b.cli_id IS NULL, что гораздо производительнее, чем NOT IN.

⸻

✅ Финальная версия запроса (по всем клиентам, кроме “особых” с НС май–июнь)

WITH excluded_clients AS (
    SELECT DISTINCT cli_id
    FROM ALM.vw_balance_rest_all WITH (NOLOCK)
    WHERE 
        dt_rep = '2025-07-30'
        AND section_name = 'Накопительный счёт'
        AND od_flag = 1
        AND MONTH(dt_open) IN (5, 6)
),

target_clients AS (
    SELECT DISTINCT a.cli_id
    FROM ALM.vw_balance_rest_all a WITH (NOLOCK)
    LEFT JOIN excluded_clients b ON a.cli_id = b.cli_id
    WHERE 
        a.dt_rep = '2025-07-30'
        AND a.od_flag = 1
        AND b.cli_id IS NULL
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

📌 Что делает запрос:
	•	Сначала находит excluded_clients — это те, кто имел НС, открытый в мае или июне по состоянию на 30.07;
	•	Затем выбирает target_clients — все остальные, через LEFT JOIN ... IS NULL;
	•	Строит срез по 30.07 и 02.08;
	•	Делит НС по prod_id;
	•	Выводит полную таблицу: объёмы, ставки, срочность, дельты.

⸻

Если нужно:
	•	включить долю prod_id в НС;
	•	агрегировать по TSegmentName, rate_trf или другим полям — могу доработать.
