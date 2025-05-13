/* агрегируем остаток по сроку (MATUR) + категория «бессрочно» */
WITH src AS (
    SELECT
        /* если хотя бы одно из трёх полей = 4444-01-01 → бессрочно */
        CASE 
            WHEN DT_CLOSE       = '4444-01-01'
              OR DT_CLOSE_FACT  = '4444-01-01'
              OR DT_CLOSE_PLAN  = '4444-01-01'
            THEN  N'бессрочно'
            ELSE  CAST(DATEDIFF(DAY, DT_OPEN_fact, DT_CLOSE) AS varchar(20))
        END                                           AS MATUR,      -- срок как строка
        OUT_RUB                                       AS out_rub     -- сумма в рублях
    FROM  ALM.dbo.balance_rest_all WITH (NOLOCK)
    WHERE MAP_IS_CASH   = 1
      AND TSEGMENTNAME  IN (N'ДЧБО', N'Розничный бизнес')
      AND AP            = N'Пассив'
      AND BLOCK_NAME    = N'Привлечение ФЛ'
      AND SECTION_NAME  IN (N'Срочные', N'До востребования', N'Накопительный счёт')
      AND DT_REP        = '2025-05-11'
)

SELECT
    MATUR,                                -- срок в днях или «бессрочно»
    SUM(out_rub)/1e6  AS total_mln        -- итого, млн руб
FROM src
GROUP BY MATUR
ORDER BY
    /* сначала числовые сроки, потом «бессрочно» */
    CASE 
        WHEN TRY_CAST(MATUR AS int) IS NULL THEN 999999
        ELSE TRY_CAST(MATUR AS int)
    END;
