WITH base AS (
    SELECT 
        DT_REP,
        CASE 
            WHEN ADDEND_NAME = 'Средства ФЛ' THEN 'Средства ФЛ'
            ELSE 'ЮЛ'
        END AS ADDEND_NAME,
        SUM(AMOUNT_RUB_MOD) AS AMOUNT_SUM
    FROM [LIQUIDITY].[ratio].[VW_SH_Ratio_Agg_LVL2_Fact]
    WHERE 
        OrganizationName = 'Банк ДОМ.РФ'
        AND ADDEND_TYPE = 'Оттоки'
        AND ADDEND_DESCR != 'Аккредитивы'
        AND ADDEND_NAME IN (
            'Средства ФЛ',
            'Средства ЮЛ',
            'Средства Ф/О в рамках станд. продуктов',
            'Средства Ф/О'
        )
    GROUP BY 
        DT_REP,
        CASE 
            WHEN ADDEND_NAME = 'Средства ФЛ' THEN 'Средства ФЛ'
            ELSE 'ЮЛ'
        END
)
SELECT 
    b.*,
    j.other_column -- подставишь своё
FROM base b
LEFT JOIN (
    SELECT 
        DT_REP, 
        other_column -- подставишь своё
    FROM your_other_table_or_subquery
    WHERE ... -- твои фильтры
) j ON b.DT_REP = j.DT_REP
ORDER BY b.DT_REP;
