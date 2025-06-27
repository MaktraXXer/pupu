/*---------------------------------------------------------------
   Договоры, открытые с 01-янв-2025 по 24-июн-2025,
   только без-опционных (IS_OPTION = 0).

   Сравниваем «ТС с восстановленной надбавкой» (старая витрина)
   и «ТС AS IS» (новая витрина); выводим разницу, флаг |Δ|>3 %
   и признак «ГГГГ-ММ» для удобной группировки по месяцам-годам.
----------------------------------------------------------------*/
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

    /* 4. Разница (п.п.) */
    CAST( n.MonthlyCONV_ALM_TransfertRate
        - s.MonthlyCONV_ALM_TransfertRate  AS DECIMAL(18,8) )    AS [Δ ТС, п.п.],

    /* 5. Флаг, если |Δ| > 3 %  (0.03 = 3 п.п. при десятичной записи процентов) */
    CASE
        WHEN ABS(n.MonthlyCONV_ALM_TransfertRate
               - s.MonthlyCONV_ALM_TransfertRate) > 0.03
             THEN 1 ELSE 0
    END                                         AS [Flag_|Δ|>3%],

    /* 6. Год-месяц открытия депозита (формат YYYY-MM) */
    CONVERT(char(7), s.DT_OPEN, 126)            AS [Месяц-Год_Открытия]

FROM ALM_TEST.WORK.DepositInterestsRateSnap      s          -- «восстановл. надбавка»
JOIN ALM_TEST.WORK.DepositInterestsRateSnap_NEW  n          -- «AS IS»
      ON n.CON_ID = s.CON_ID                     -- 1-к-1 совпадение договоров

WHERE s.DT_OPEN BETWEEN '2025-01-01' AND '2025-06-24'  -- пункт 1
  AND s.IS_OPTION = 0                                  -- пункт 1.5
  AND n.IS_OPTION = 0                                  -- симметричная фильтрация

ORDER BY s.DT_OPEN, s.CON_ID;                          -- удобный порядок вывода
