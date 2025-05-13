/* -------- остаток по remaining life -------- */
WITH src AS (
    SELECT
        /* выбираем «живую» дату закрытия (!= 4444-01-01) */
        COALESCE(
            NULLIF(DT_CLOSE_FACT , '4444-01-01'),
            NULLIF(DT_CLOSE_PLAN, '4444-01-01'),
            NULLIF(DT_CLOSE     , '4444-01-01')
        )                                           AS real_close,
        DT_REP,
        OUT_RUB
    FROM  ALM.dbo.balance_rest_all WITH (NOLOCK)
    WHERE MAP_IS_CASH   = 1
      AND TSEGMENTNAME  IN (N'ДЧБО', N'Розничный бизнес')
      AND AP            = N'Пассив'
      AND BLOCK_NAME    = N'Привлечение ФЛ'
      AND SECTION_NAME  IN (N'Срочные', N'До востребования', N'Накопительный счёт')
      AND DT_REP        = '2025-05-11'
)

/* -------- «сколько жить осталось» -------- */
, lifetable AS (
    SELECT
        CASE
             WHEN real_close IS NULL
                  THEN N'бессрочно'                                -- нет даты или 4444-01-01
             ELSE CAST(DATEDIFF(DAY, DT_REP, real_close) AS varchar(20))
        END                                        AS days_left,
        OUT_RUB
    FROM src
)

SELECT
    days_left                               AS [дней до погашения],
    SUM(OUT_RUB) / 1e6  AS total_mln        -- сумма в млн руб
FROM lifetable
GROUP BY days_left
ORDER BY
    CASE
        WHEN TRY_CAST(days_left AS int) IS NULL THEN  999999   -- «бессрочно» внизу
        ELSE TRY_CAST(days_left AS int)
    END;
