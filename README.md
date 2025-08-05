Отлично, вот полный SQL-запрос, который:

⸻

✅ Делает всё в одной таблице:
	•	Берёт только тех cli_id, у кого на 30.07 был НС, открытый в мае-июне;
	•	Считает по этим клиентам портфель в разрезе:
	•	на 30.07 и на 02.08;
	•	отдельно по НС и срочным вкладам;
	•	Строит дельты (разницы) по:
	•	объёму;
	•	ставке;
	•	срочности (по вкладам).

⸻

✅ Полный SQL-запрос

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

agg_0730 AS (
    SELECT
        -- объёмы
        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) AS ns_out_rub_0730,
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) AS dep_out_rub_0730,

        -- средневзвешенная ставка НС
        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB * con_rate ELSE 0 END) AS ns_weighted_rate_0730,
        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) AS ns_base_0730,

        -- средневзвешенная ставка и срочность по вкладам
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * con_rate ELSE 0 END) AS dep_weighted_rate_0730,
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * termdays ELSE 0 END) AS dep_weighted_term_0730,
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) AS dep_base_0730
    FROM data_0730
),

agg_0208 AS (
    SELECT
        -- объёмы
        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) AS ns_out_rub_0208,
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) AS dep_out_rub_0208,

        -- средневзвешенная ставка НС
        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB * con_rate ELSE 0 END) AS ns_weighted_rate_0208,
        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN OUT_RUB ELSE 0 END) AS ns_base_0208,

        -- средневзвешенная ставка и срочность по вкладам
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * con_rate ELSE 0 END) AS dep_weighted_rate_0208,
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB * termdays ELSE 0 END) AS dep_weighted_term_0208,
        SUM(CASE WHEN section_name = 'Срочные' THEN OUT_RUB ELSE 0 END) AS dep_base_0208
    FROM data_0208
)

SELECT
    -- ========== ОБЪЕМЫ ==========
    a.ns_out_rub_0730,
    b.ns_out_rub_0208,
    b.ns_out_rub_0208 - a.ns_out_rub_0730 AS delta_ns_out_rub,

    a.dep_out_rub_0730,
    b.dep_out_rub_0208,
    b.dep_out_rub_0208 - a.dep_out_rub_0730 AS delta_dep_out_rub,

    -- ========== СТАВКИ ==========
    CASE 
        WHEN a.ns_base_0730 > 0 THEN a.ns_weighted_rate_0730 / a.ns_base_0730
        ELSE NULL
    END AS ns_avg_rate_0730,

    CASE 
        WHEN b.ns_base_0208 > 0 THEN b.ns_weighted_rate_0208 / b.ns_base_0208
        ELSE NULL
    END AS ns_avg_rate_0208,

    CASE 
        WHEN a.ns_base_0730 > 0 AND b.ns_base_0208 > 0 THEN 
            (b.ns_weighted_rate_0208 / b.ns_base_0208) - (a.ns_weighted_rate_0730 / a.ns_base_0730)
        ELSE NULL
    END AS delta_ns_rate,

    CASE 
        WHEN a.dep_base_0730 > 0 THEN a.dep_weighted_rate_0730 / a.dep_base_0730
        ELSE NULL
    END AS dep_avg_rate_0730,

    CASE 
        WHEN b.dep_base_0208 > 0 THEN b.dep_weighted_rate_0208 / b.dep_base_0208
        ELSE NULL
    END AS dep_avg_rate_0208,

    CASE 
        WHEN a.dep_base_0730 > 0 AND b.dep_base_0208 > 0 THEN 
            (b.dep_weighted_rate_0208 / b.dep_base_0208) - (a.dep_weighted_rate_0730 / a.dep_base_0730)
        ELSE NULL
    END AS delta_dep_rate,

    -- ========== СРОЧНОСТЬ ==========
    CASE 
        WHEN a.dep_base_0730 > 0 THEN a.dep_weighted_term_0730 / a.dep_base_0730
        ELSE NULL
    END AS dep_avg_term_0730,

    CASE 
        WHEN b.dep_base_0208 > 0 THEN b.dep_weighted_term_0208 / b.dep_base_0208
        ELSE NULL
    END AS dep_avg_term_0208,

    CASE 
        WHEN a.dep_base_0730 > 0 AND b.dep_base_0208 > 0 THEN 
            (b.dep_weighted_term_0208 / b.dep_base_0208) - (a.dep_weighted_term_0730 / a.dep_base_0730)
        ELSE NULL
    END AS delta_dep_term

FROM agg_0730 a
JOIN agg_0208 b ON 1=1


⸻

🧾 Что покажет результат:

Показатель	Назначение
ns_out_rub_0730 / 0208	общий объём на НС на даты
dep_out_rub_0730 / 0208	общий объём по вкладам
ns_avg_rate_0730 / 0208	средневзв. ставка по НС
dep_avg_rate_0730 / 0208	средневзв. ставка по вкладам
dep_avg_term_0730 / 0208	средневзв. срочность вкладов
delta_*	разницы по всем показателям


⸻

Если тебе нужно это в разбивке по cli_id — можно адаптировать. Сейчас расчёт по всему портфелю (в рамках target_clients).
