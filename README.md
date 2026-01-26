DECLARE @Cur NVARCHAR(100) = '810';   -- '810' / 'all' / '840' / ...
DECLARE @dt_old DATE = '2025-10-01';
DECLARE @dt_new DATE = '2025-10-05';
DECLARE @product NVARCHAR(100) = 'SA'; -- 'SA' / 'deposit' / 'all'
DECLARE @specific_cli_id NVARCHAR(100) = '0'; -- '0' if no specific client
;

WITH
old AS(
    SELECT
        dt_rep,
        cli_id,
        SUM(CASE
                WHEN @Cur IN ('810','all') THEN OUT_RUB
                ELSE OUT_CUR
            END)/1000000.0 AS [всего_остаток_млн__старое],

        SUM(CASE WHEN section_name = 'Срочные' THEN
            CASE
                WHEN @Cur IN ('810','all') THEN OUT_RUB
                ELSE OUT_CUR
            END END)/1000000.0 AS [Вклады_млн__старое],

        SUM(CASE WHEN section_name = 'До востребования' THEN
            CASE
                WHEN @Cur IN ('810','all') THEN OUT_RUB
                ELSE OUT_CUR
            END END)/1000000.0 AS [ДВС_млн__старое],

        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN
            CASE
                WHEN @Cur IN ('810','all') THEN OUT_RUB
                ELSE OUT_CUR
            END END)/1000000.0 AS [Накоп_счет_млн__старое]
    FROM [ALM].[ALM].[balance_rest_all] WITH (NOLOCK)
    WHERE dt_rep = @dt_old
      AND od_flag = 1
      AND block_name = 'Привлечение ФЛ'
      AND SECTION_NAME NOT IN ('Аккредитивы', 'Аккредитив под строительство', 'Брокерское обслуживание')
      AND (@Cur = 'all' OR CUR = @Cur)  -- если all, фильтра нет; иначе фильтр по CUR
    GROUP BY dt_rep, cli_id
),
new AS(
    SELECT
        dt_rep,
        cli_id,
        SUM(CASE
                WHEN @Cur IN ('810','all') THEN OUT_RUB
                ELSE OUT_CUR
            END)/1000000.0 AS [всего_остаток_млн__новое],

        SUM(CASE WHEN section_name = 'Срочные' THEN
            CASE
                WHEN @Cur IN ('810','all') THEN OUT_RUB
                ELSE OUT_CUR
            END END)/1000000.0 AS [Вклады_млн__новое],

        SUM(CASE WHEN section_name = 'До востребования' THEN
            CASE
                WHEN @Cur IN ('810','all') THEN OUT_RUB
                ELSE OUT_CUR
            END END)/1000000.0 AS [ДВС_млн__новое],

        SUM(CASE WHEN section_name = 'Накопительный счёт' THEN
            CASE
                WHEN @Cur IN ('810','all') THEN OUT_RUB
                ELSE OUT_CUR
            END END)/1000000.0 AS [Накоп_счет_млн__новое]
    FROM [ALM].[ALM].[balance_rest_all] WITH (NOLOCK)
    WHERE dt_rep = @dt_new
      AND od_flag = 1
      AND block_name = 'Привлечение ФЛ'
      AND SECTION_NAME NOT IN ('Аккредитивы', 'Аккредитив под строительство', 'Брокерское обслуживание')
      AND (@Cur = 'all' OR CUR = @Cur)
    GROUP BY dt_rep, cli_id
)
SELECT
    COALESCE(n.cli_id, o.cli_id) AS cli_id,
    o.всего_остаток_млн__старое,
    n.всего_остаток_млн__новое,
    n.всего_остаток_млн__новое - COALESCE(o.всего_остаток_млн__старое, 0) AS [Дельта_общий_остаток],

    o.Вклады_млн__старое,
    n.Вклады_млн__новое,
    n.Вклады_млн__новое - COALESCE(o.Вклады_млн__старое, 0) AS [Дельта_вклады],

    o.ДВС_млн__старое,
    n.ДВС_млн__новое,
    n.ДВС_млн__новое - COALESCE(o.ДВС_млн__старое, 0) AS [Дельта_ДВС],

    o.Накоп_счет_млн__старое,
    n.Накоп_счет_млн__новое,
    n.Накоп_счет_млн__новое - COALESCE(o.Накоп_счет_млн__старое, 0) AS [Дельта_накоп]
FROM new n
FULL OUTER JOIN old o
    ON n.cli_id = o.cli_id
WHERE
    (
      (CASE WHEN @product = 'SA' THEN n.Накоп_счет_млн__новое
            WHEN @product = 'deposit' THEN n.Вклады_млн__новое
            WHEN @product = 'all' THEN n.всего_остаток_млн__новое END) IS NOT NULL
      OR
      (CASE WHEN @product = 'SA' THEN o.Накоп_счет_млн__старое
            WHEN @product = 'deposit' THEN o.Вклады_млн__старое
            WHEN @product = 'all' THEN o.всего_остаток_млн__старое END) IS NOT NULL
    )
    AND COALESCE(n.cli_id, o.cli_id) = CASE WHEN @specific_cli_id = '0'
                                           THEN COALESCE(n.cli_id, o.cli_id)
                                           ELSE @specific_cli_id END
ORDER BY
    ABS(
        (CASE WHEN @product = 'SA' THEN n.Накоп_счет_млн__новое
              WHEN @product = 'deposit' THEN n.Вклады_млн__новое
              WHEN @product = 'all' THEN n.всего_остаток_млн__новое END)
        - COALESCE(
            (CASE WHEN @product = 'SA' THEN o.Накоп_счет_млн__старое
                  WHEN @product = 'deposit' THEN o.Вклады_млн__старое
                  WHEN @product = 'all' THEN o.всего_остаток_млн__старое END), 0
        )
    ) DESC
OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY;
