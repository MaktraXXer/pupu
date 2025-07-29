SELECT 
    CAST(dt_rep AS date) AS dt_rep,
    SUM(CASE WHEN block_name = 'Привлечение ЮЛ' THEN sout_rub ELSE 0 END) AS [ЮЛ],
    SUM(CASE WHEN block_name = 'Привлечение ФЛ' THEN sout_rub ELSE 0 END) AS [ФЛ]
FROM ALM.ALM.VW_alm_balance_AGG WITH (NOLOCK)
WHERE 
    is_cash = 1
    AND isnull(is_floatRate, '0') = '0'
    AND isnull([Л> Признак], 'Аукционы') IN ('Аукционы', 'Регионы')
    AND ap = 'Пассив'
    AND block_name IN ('Привлечение ЮЛ', 'Привлечение ФЛ')
    AND section_name = 'Срочные'
    AND dt_rep >= '2025-01-01'
GROUP BY CAST(dt_rep AS date)
ORDER BY dt_rep
