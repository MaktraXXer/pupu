/* -------- остаток по remaining life -------- */
WITH src AS (
    SELECT
        /* выбираем реальную дату закрытия: фактическая, плановая, расчётная */
        ISNULL(NULLIF(DT_CLOSE_FACT,'4444-01-01'),
               NULLIF(DT_CLOSE_PLAN,'4444-01-01'),
               NULLIF(DT_CLOSE     ,'4444-01-01'))           AS real_close,
        dt_rep,
        out_rub
    FROM  ALM.dbo.balance_rest_all WITH (NOLOCK)
    WHERE MAP_IS_CASH   = 1
      AND TSEGMENTNAME  IN (N'ДЧБО', N'Розничный бизнес')
      AND AP            = N'Пассив'
      AND BLOCK_NAME    = N'Привлечение ФЛ'
      AND SECTION_NAME  IN (N'Срочные', N'До востребования', N'Накопительный счёт')
      AND DT_REP        = '2025-05-11'
)
/* -------- рассчитываем дней до выхода -------- */
, lifetable AS (
    SELECT
        CASE
             /* бессрочный: нет даты или спец-значение 4444-01-01 */
             WHEN real_close IS NULL THEN N'бессрочно'
             WHEN real_close = '4444-01-01' THEN N'бессрочно'
             /* иначе — число дней до погашения */
             ELSE CAST(DATEDIFF(DAY, dt_rep, real_close) AS varchar(20))
        END            AS days_left,
        out_rub
    FROM src
)
SELECT
    days_left                          AS [дней до погашения],
    SUM(out_rub)/1e6   AS total_mln    -- итог в млн руб
FROM lifetable
GROUP BY days_left
ORDER BY
    CASE
        WHEN TRY_CAST(days_left AS int) IS NULL THEN  999999   -- «бессрочно» внизу
        ELSE TRY_CAST(days_left AS int)
    END;
