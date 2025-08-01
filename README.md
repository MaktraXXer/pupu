Отлично, ты прав — наиболее надёжный способ определить закрытие вклада — это сравнить snapshot’ы баланса на 30.06 и 30.07 и выявить con_id, которые были в июне, но исчезли к июлю.

⸻

✅ Новый корректный подход:

⸻

🔧 1. Выгрузим con_id на 30.06.2025:

SELECT DISTINCT t.con_id, t.cli_id, t.out_rub
INTO #balance_june
FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep = '2025-06-30'
  AND t.section_name = N'Срочные'
  AND t.is_floatrate = 0
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.OUT_RUB IS NOT NULL


⸻

🔧 2. Выгрузим con_id на 30.07.2025:

SELECT DISTINCT t.con_id
INTO #balance_july
FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep = '2025-07-30'
  AND t.section_name = N'Срочные'
  AND t.is_floatrate = 0
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.OUT_RUB IS NOT NULL


⸻

🔧 3. Найдём закрытые con_id и сгруппируем по клиенту:

WITH closed_accounts AS (
    SELECT b.con_id, b.cli_id, b.out_rub
    FROM #balance_june b
    LEFT JOIN #balance_july j
           ON b.con_id = j.con_id
    WHERE j.con_id IS NULL  -- не найден во втором срезе
)
, grouped AS (
    SELECT
        cli_id,
        SUM(out_rub) AS total_out_rub,
        STRING_AGG(CAST(con_id AS NVARCHAR), ',') AS con_id_list
    FROM closed_accounts
    GROUP BY cli_id
)
SELECT 
    g.cli_id,
    g.total_out_rub,
    g.con_id_list,
    t.*
FROM grouped g
JOIN ALM.[ehd].[VW_transfers_FL_det] t WITH (NOLOCK)
  ON g.cli_id = t.cli_id
WHERE t.dt_rep BETWEEN '2025-07-01' AND '2025-07-31'


⸻

📌 Результат:
	•	Учитываются только con_id, исчезнувшие между 30.06 и 30.07
	•	Получаем cli_id, сумму вкладов, список счетов
	•	Добавляем к ним переводы в июле

⸻

💡 Дополнительно:

Если хочешь — можно также проверить saldo для этих con_id, чтобы убедиться, что их реально не существует на дату — но основная логика уже корректна, без риска пропустить нулевые остатки или неточные интервалы.

Готов также переписать всё в одном SQL-блоке без временных таблиц, если нужно.
