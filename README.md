
USE ALM;
SET NOCOUNT ON;

DECLARE @dt date = '2026-09-01';

DROP TABLE IF EXISTS #diag;

SELECT
    t.block_name,
    t.section_name,
    t.od_flag,
    t.cur,

    COUNT_BIG(*) AS rows_all,

    COUNT_BIG(CASE
        WHEN t.CLI_ID IS NOT NULL
        THEN 1 END) AS rows_cli,

    COUNT_BIG(CASE
        WHEN t.CLI_ID IS NOT NULL
         AND t.out_rub >= 0
        THEN 1 END) AS rows_balance,

    COUNT_BIG(CASE
        WHEN t.CLI_ID IS NOT NULL
         AND t.out_rub >= 0
         AND t.PROD_NAME_res IS NOT NULL
        THEN 1 END) AS rows_product_not_null,

    COUNT_BIG(CASE
        WHEN t.CLI_ID IS NOT NULL
         AND t.out_rub >= 0
         AND t.PROD_NAME_res NOT IN (
            N'Надёжный прайм',
            N'Надёжный VIP',
            N'Надёжный премиум',
            N'Надёжный промо',
            N'Надёжный старт',
            N'Надёжный Т2',
            N'Надёжный Мегафон',
            N'Надёжный процент',
            N'Могучий',
            N'Надёжный',
            N'ДОМанадёжно',
            N'Всё в ДОМ'
         )
        THEN 1 END) AS rows_filtered,

    COUNT_BIG(CASE
        WHEN t.CLI_ID IS NOT NULL
         AND t.out_rub > 0
         AND t.PROD_NAME_res NOT IN (
            N'Надёжный прайм',
            N'Надёжный VIP',
            N'Надёжный премиум',
            N'Надёжный промо',
            N'Надёжный старт',
            N'Надёжный Т2',
            N'Надёжный Мегафон',
            N'Надёжный процент',
            N'Могучий',
            N'Надёжный',
            N'ДОМанадёжно',
            N'Всё в ДОМ'
         )
        THEN 1 END) AS rows_positive

INTO #diag

FROM ALM.ALM.balance_rest_all t WITH (NOLOCK)

WHERE t.dt_rep >= @dt
  AND t.dt_rep < DATEADD(day, 1, @dt)

GROUP BY
    t.block_name,
    t.section_name,
    t.od_flag,
    t.cur

OPTION (MAXDOP 1, RECOMPILE);


/* Результат 1: последовательная проверка фильтров */

;WITH s AS (
    SELECT
        ISNULL(SUM(rows_all), 0) AS s0,

        ISNULL(SUM(CASE
            WHEN block_name = N'Привлечение ФЛ'
            THEN rows_all ELSE 0 END), 0) AS s1,

        ISNULL(SUM(CASE
            WHEN block_name = N'Привлечение ФЛ'
             AND section_name IN (
                N'Срочные',
                N'Накопительный счёт',
                N'До востребования'
             )
            THEN rows_all ELSE 0 END), 0) AS s2,

        ISNULL(SUM(CASE
            WHEN block_name = N'Привлечение ФЛ'
             AND section_name IN (
                N'Срочные',
                N'Накопительный счёт',
                N'До востребования'
             )
             AND od_flag = 1
            THEN rows_all ELSE 0 END), 0) AS s3,

        ISNULL(SUM(CASE
            WHEN block_name = N'Привлечение ФЛ'
             AND section_name IN (
                N'Срочные',
                N'Накопительный счёт',
                N'До востребования'
             )
             AND od_flag = 1
             AND cur = '810'
            THEN rows_all ELSE 0 END), 0) AS s4,

        ISNULL(SUM(CASE
            WHEN block_name = N'Привлечение ФЛ'
             AND section_name IN (
                N'Срочные',
                N'Накопительный счёт',
                N'До востребования'
             )
             AND od_flag = 1
             AND cur = '810'
            THEN rows_cli ELSE 0 END), 0) AS s5,

        ISNULL(SUM(CASE
            WHEN block_name = N'Привлечение ФЛ'
             AND section_name IN (
                N'Срочные',
                N'Накопительный счёт',
                N'До востребования'
             )
             AND od_flag = 1
             AND cur = '810'
            THEN rows_balance ELSE 0 END), 0) AS s6,

        ISNULL(SUM(CASE
            WHEN block_name = N'Привлечение ФЛ'
             AND section_name IN (
                N'Срочные',
                N'Накопительный счёт',
                N'До востребования'
             )
             AND od_flag = 1
             AND cur = '810'
            THEN rows_filtered ELSE 0 END), 0) AS s7,

        ISNULL(SUM(CASE
            WHEN block_name = N'Привлечение ФЛ'
             AND section_name IN (
                N'Срочные',
                N'Накопительный счёт',
                N'До востребования'
             )
             AND od_flag = 1
             AND cur = '810'
            THEN rows_positive ELSE 0 END), 0) AS s8

    FROM #diag
)

SELECT
    v.step_name,
    v.rows_count

FROM s
CROSS APPLY (VALUES
    (0, N'Все строки за 01.09', s0),
    (1, N'После block_name', s1),
    (2, N'После section_name', s2),
    (3, N'После od_flag = 1', s3),
    (4, N'После cur = 810', s4),
    (5, N'После CLI_ID IS NOT NULL', s5),
    (6, N'После out_rub >= 0', s6),
    (7, N'После исключения продуктов', s7),
    (8, N'Из них с положительным out_rub', s8)
) v(step_id, step_name, rows_count)

ORDER BY v.step_id;


/* Результат 2: какие значения реально есть в балансе */

SELECT
    block_name,
    section_name,
    od_flag,
    cur,
    rows_all,
    rows_cli,
    rows_balance,
    rows_product_not_null,
    rows_filtered,
    rows_positive

FROM #diag

ORDER BY rows_all DESC;
