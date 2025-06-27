/*---------------------------------------------------------------------------
   Сравнение «ТС с восстановленной надбавкой» и «ТС AS IS»
   ─ только договоры, открытые 01-янв-2025 – 24-июн-2025
   ─ только без-опционных (IS_OPTION = 0)
   ─ флаг > 3 % рассчитывается по **относительному** отклонению
---------------------------------------------------------------------------*/
SELECT
    s.CON_ID                               AS [CON_ID],
    s.DT_OPEN                              AS [DT_OPEN],
    s.DT_CLOSE                             AS [DT_CLOSE],
    s.BALANCE_RUB                          AS [BALANCE_RUB],
    s.CONVENTION                           AS [CONVENTION],
    s.PROD_NAME                            AS [PROD_NAME],

    /* 2. ТС с восстановленной надбавкой */
    s.MonthlyCONV_ALM_TransfertRate        AS [ТС с восстановленной надбавкой],

    /* 3. ТС AS IS */
    n.MonthlyCONV_ALM_TransfertRate        AS [ТС AS IS],

    /* 4. Абс. разница (п.п.) — оставим, может пригодиться */
    CAST( n.MonthlyCONV_ALM_TransfertRate
        - s.MonthlyCONV_ALM_TransfertRate  AS DECIMAL(18,8) )    AS [Δ ТС, п.п.],

    /* 4b. Относительная разница (доля, напр. 0.031 = 3.1 %) */
    CAST(
        ( n.MonthlyCONV_ALM_TransfertRate - s.MonthlyCONV_ALM_TransfertRate )
        / NULLIF(ABS(s.MonthlyCONV_ALM_TransfertRate),0)
        AS DECIMAL(18,8)
    )                                        AS [Δ ТС, относит.],

    /* 5. Флаг: |относ. Δ| > 3 %  → 1, иначе 0  */
    CASE
        WHEN ABS(
               ( n.MonthlyCONV_ALM_TransfertRate - s.MonthlyCONV_ALM_TransfertRate )
               / NULLIF(ABS(s.MonthlyCONV_ALM_TransfertRate),0)
             ) > 0.03
             THEN 1 ELSE 0
    END                                       AS [Flag_|Δ|>3%],

    /* 6. Год-месяц открытия (YYYY-MM) */
    CONVERT(char(7), s.DT_OPEN, 126)          AS [Месяц-Год_Открытия]

FROM ALM_TEST.WORK.DepositInterestsRateSnap      s
JOIN ALM_TEST.WORK.DepositInterestsRateSnap_NEW  n
      ON n.CON_ID = s.CON_ID                     -- пары 1-к-1

WHERE s.DT_OPEN BETWEEN '2025-01-01' AND '2025-06-24'
  AND s.IS_OPTION = 0                            -- пункт 1.5
  AND n.IS_OPTION = 0

ORDER BY s.DT_OPEN, s.CON_ID;
