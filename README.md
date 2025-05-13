/* === агрегируем остаток по сроку MATUR === */
WITH src AS (
    SELECT
        DATEDIFF(DAY, DT_OPEN_fact, DT_CLOSE) AS MATUR,     -- срок, дней
        OUT_RUB                                AS out_rub   -- можно заменить на BALANCE_RUB, если колонка есть
    FROM  ALM.dbo.balance_rest_all WITH (NOLOCK)
    WHERE MAP_IS_CASH   = 1
      AND TSEGMENTNAME  IN (N'ДЧБО', N'Розничный бизнес')
      AND AP            = N'Пассив'
      AND BLOCK_NAME    = N'Привлечение ФЛ'
      AND SECTION_NAME  IN (N'Срочные', N'До востребования', N'Накопительный счёт')
      AND DT_REP        = '2025-05-11'
)
SELECT
    MATUR,                                   -- срок в днях
    SUM(out_rub)        / 1e6 AS total_mln   -- итог в млн руб (уберите деление, если нужен в рублях)
FROM src
GROUP BY MATUR
ORDER BY MATUR;                              -- по возрастанию
