/* ---------- подготовка ---------- */
WITH src AS (
    SELECT
        remainingdays,          -- готовое количество дней до погашения
        DT_CLOSE,
        DT_CLOSE_FACT,
        DT_CLOSE_PLAN,
        OUT_RUB
    FROM  ALM.dbo.balance_rest_all WITH (NOLOCK)
    WHERE MAP_IS_CASH   = 1
      AND TSEGMENTNAME  IN (N'ДЧБО', N'Розничный бизнес')
      AND AP            = N'Пассив'
      AND BLOCK_NAME    = N'Привлечение ФЛ'
      AND SECTION_NAME  IN (N'Срочные', N'До востребования', N'Накопительный счёт')
      AND DT_REP        = '2025-05-11'
)

/* ---------- остаток по remainingdays ---------- */
, lifetable AS (
    SELECT
        CASE
            /* “бессрочно”, если хотя бы одна из дат-заглушек = 4444-01-01
               или remainingdays = NULL / очень большое (подстраховка) */
            WHEN DT_CLOSE      = '4444-01-01'
              OR DT_CLOSE_FACT = '4444-01-01'
              OR DT_CLOSE_PLAN = '4444-01-01'
              OR remainingdays IS NULL
            THEN N'бессрочно'

            /* “просрочено”, если осталось < 0 дней */
            WHEN remainingdays < 0
            THEN N'просрочено'

            /* иначе — число дней до погашения */
            ELSE CAST(remainingdays AS varchar(20))
        END              AS days_left,
        OUT_RUB
    FROM src
)

/* ---------- итог ---------- */
SELECT
    days_left                          AS [дней до погашения],
    SUM(out_rub)/1e6  AS total_mln     -- сумма, млн руб
FROM lifetable
GROUP BY days_left
ORDER BY
    CASE
        WHEN days_left = N'просрочено' THEN  999998
        WHEN days_left = N'бессрочно'  THEN  999999
        ELSE TRY_CAST(days_left AS int)
    END;
