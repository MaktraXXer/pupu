Отлично, давай я помогу тебе собрать оба запроса — коротко, аккуратно и так, чтобы они реально давали удобные агрегаты для выгрузки в Excel.

⸻

🔹 ЗАДАЧА 1

(остатки и средневзвешенная ставка на нужные даты)

👉 Нам нужны 4 даты:
	•	конец прошлого года
	•	EOMONTH(GETDATE(),-1)-2 (ровно месяц назад минус 2 дня)
	•	GETDATE()-9
	•	GETDATE()-2

SELECT 
    dt_rep,
    SUM(out_rub)                           AS total_out_rub,
    SUM(out_rub * rate_trf)/SUM(out_rub)   AS w_avg_rate
FROM ALM.VW_Balance_Rest_All WITH (NOLOCK)
WHERE section_name = 'Прочие'
  AND TPROD_NAME  = 'Кредит наличными'
  AND cur         = '810'
  AND BLOCK_NAME  = 'Кредиты ФЛ'
  AND od_flag     = 1
  AND OUT_RUB IS NOT NULL
  AND dt_rep IN (
        EOMONTH(DATEADD(YEAR,-1,GETDATE())),        -- конец прошлого года
        DATEADD(DAY,-2,EOMONTH(GETDATE(),-1)),      -- месяц назад - 2 дня
        CAST(DATEADD(DAY,-9,GETDATE()) AS date),    -- 9 дней назад
        CAST(DATEADD(DAY,-2,GETDATE()) AS date)     -- 2 дня назад
  )
GROUP BY dt_rep
ORDER BY dt_rep;


⸻

🔹 ЗАДАЧА 2

(новые выдачи за последние 6 дней, dt_open ∈ [getdate()-8 ; getdate()-2])

👉 Нужно:
	•	средневзвешенный срок (termdays)
	•	средневзвешенная ставка (rate_con)
	•	объем в 2 категориях (rate_trf >= 0.15 и <0.15)

SELECT
    CAST(GETDATE()-2 AS date)                          AS dt_rep,
    SUM(OUT_RUB)                                       AS total_out_rub,
    SUM(OUT_RUB * termdays) / NULLIF(SUM(OUT_RUB),0)   AS w_avg_termdays,
    SUM(OUT_RUB * rate_con) / NULLIF(SUM(OUT_RUB),0)   AS w_avg_rate_con,
    SUM(CASE WHEN rate_trf >= 0.15 THEN OUT_RUB ELSE 0 END) AS vol_high_trf,
    SUM(CASE WHEN rate_trf <  0.15 THEN OUT_RUB ELSE 0 END) AS vol_low_trf
FROM ALM.VW_Balance_Rest_All WITH (NOLOCK)
WHERE dt_rep      = CAST(GETDATE()-2 AS date)
  AND section_name = 'Прочие'
  AND TPROD_NAME   = 'Кредит наличными'
  AND cur          = '810'
  AND BLOCK_NAME   = 'Кредиты ФЛ'
  AND od_flag      = 1
  AND OUT_RUB IS NOT NULL
  AND DT_OPEN BETWEEN CAST(GETDATE()-8 AS date) AND CAST(GETDATE()-2 AS date);


⸻

✅ Оба запроса теперь:
	•	короткие, без «1=1» и мусора;
	•	дают аккуратную агрегированную таблицу (можно вставлять в Excel и обновлять);
	•	в Задаче 2 сразу есть всё: объём, средневзвешенный срок, ставка и разбивка по категориям.

⸻

Хочешь, я соберу их ещё в одну процедуру, чтобы было прям «одной кнопкой» и Excel всегда тянул оба результата?
