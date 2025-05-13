SELECT
    days_left,                       -- дней до погашения / «бессрочно» / «просрочено»
    SUM(out_rub)/1e6 AS total_mln    -- сумма, млн руб.
FROM (
    SELECT
        CASE
            WHEN DT_CLOSE      = '4444-01-01'
              OR DT_CLOSE_FACT = '4444-01-01'
              OR DT_CLOSE_PLAN = '4444-01-01'
              OR remainingdays IS NULL
                THEN N'бессрочно'          -- нет конечной даты
            WHEN remainingdays < 0
                THEN N'просрочено'         -- срок уже истёк
            ELSE CAST(remainingdays AS varchar(20))
        END           AS days_left,
        OUT_RUB       AS out_rub
    FROM  ALM.dbo.balance_rest_all WITH (NOLOCK)
    WHERE MAP_IS_CASH  = 1
      AND TSEGMENTNAME IN (N'ДЧБО', N'Розничный бизнес')
      AND AP           = N'Пассив'
      AND BLOCK_NAME   = N'Привлечение ФЛ'
      AND SECTION_NAME IN (N'Срочные', N'До востребования', N'Накопительный счёт')
      -- календарные сутки 11-го мая 2025 г.
      AND dt_rep >= '20250511'
      AND dt_rep <  '20250512'
) AS t
GROUP BY days_left
ORDER BY
    CASE
        WHEN days_left = N'просрочено' THEN 999998   -- внизу
        WHEN days_left = N'бессрочно'  THEN 999999   -- внизу
        ELSE TRY_CAST(days_left AS int)              -- 0,1,2,…
    END;
