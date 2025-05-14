/*----------------------------------------------------------
   ТОП-20 клиентов по совокупному остатку (млн руб)
   Период: 23-01-2025 … сегодня
----------------------------------------------------------*/
SELECT TOP (20)
       ROW_NUMBER() OVER (ORDER BY SUM(out_rub) DESC) AS rn,
       cli_id,
       ROUND(SUM(out_rub) / 1000000.0, 2)             AS total_for_period_mln
FROM ALM.ALM.balance_rest_all WITH (NOLOCK)
WHERE dt_rep >= '2025-01-23'                -- начало периода
  AND dt_rep <= CAST(GETDATE() AS date)     -- «сегодня»
  AND od_flag     = 1
  AND block_name  = N'Привлечение ФЛ'
  AND section_name NOT IN (N'Аккредитивы',
                           N'Аккредитив под строительство',
                           N'Брокерское обслуживание')
GROUP BY cli_id
ORDER BY SUM(out_rub) DESC                  -- ↓ ранжируем
OPTION (FAST 20);                           -- ← оптимизатору: нужны только первые 20
