DECLARE @dt_old date = '2025-09-30';
DECLARE @dt_new date = '2025-10-10';

-- 1️⃣ Клиенты с переводами себе в Т-Банк
WITH transfers AS (
    SELECT DISTINCT cli_id
    FROM ALM.[ehd].[VW_transfers_FL_det] t
    WHERE t.dt_rep BETWEEN '2025-10-01' AND '2025-10-10'
      AND t.is_self_flag = 1
      AND t.transaction_type <> N'Наличные'
      AND t.transit_max_id IS NULL
      AND t.direction_type = N'Переводы себе'
      AND t.[целевая категория] = 1
      AND t.bank_name_main = N'АО "ТБанк"'
      AND t.spec_cat = N'-'
),

-- 2️⃣ Остатки на 30 сентября
old AS (
    SELECT
        dt_rep,
        cli_id,
        SUM(CASE WHEN cur = '810' THEN OUT_RUB END) / 1e6 AS всего_остаток_млн_руб_старое,
        SUM(CASE WHEN cur = '810' AND section_name = N'Срочные' THEN OUT_RUB END) / 1e6 AS вклады_млн_руб_старое,
        SUM(CASE WHEN cur = '810' AND section_name = N'До востребования' THEN OUT_RUB END) / 1e6 AS двс_млн_руб_старое,
        SUM(CASE WHEN cur = '810' AND section_name = N'Накопительный счёт' THEN OUT_RUB END) / 1e6 AS накоп_млн_руб_старое
    FROM ALM.ALM.balance_rest_all WITH (NOLOCK)
    WHERE dt_rep = @dt_old
      AND od_flag = 1
      AND block_name = N'Привлечение ФЛ'
      AND SECTION_NAME NOT IN (N'Аккредитивы', N'Аккредитив под строительство', N'Брокерское обслуживание')
    GROUP BY dt_rep, cli_id
),

-- 3️⃣ Остатки на 10 октября
new AS (
    SELECT
        dt_rep,
        cli_id,
        SUM(CASE WHEN cur = '810' THEN OUT_RUB END) / 1e6 AS всего_остаток_млн_руб_новое,
        SUM(CASE WHEN cur = '810' AND section_name = N'Срочные' THEN OUT_RUB END) / 1e6 AS вклады_млн_руб_новое,
        SUM(CASE WHEN cur = '810' AND section_name = N'До востребования' THEN OUT_RUB END) / 1e6 AS двс_млн_руб_новое,
        SUM(CASE WHEN cur = '810' AND section_name = N'Накопительный счёт' THEN OUT_RUB END) / 1e6 AS накоп_млн_руб_новое
    FROM ALM.ALM.balance_rest_all WITH (NOLOCK)
    WHERE dt_rep = @dt_new
      AND od_flag = 1
      AND block_name = N'Привлечение ФЛ'
      AND SECTION_NAME NOT IN (N'Аккредитивы', N'Аккредитив под строительство', N'Брокерское обслуживание')
    GROUP BY dt_rep, cli_id
)

-- 4️⃣ Сводка только по клиентам с переводами
SELECT
    COALESCE(n.cli_id, o.cli_id) AS cli_id,
    o.всего_остаток_млн_руб_старое,
    n.всего_остаток_млн_руб_новое,
    n.всего_остаток_млн_руб_новое - COALESCE(o.всего_остаток_млн_руб_старое, 0) AS дельта_общий,

    o.вклады_млн_руб_старое,
    n.вклады_млн_руб_новое,
    n.вклады_млн_руб_новое - COALESCE(o.вклады_млн_руб_старое, 0) AS дельта_вклады,

    o.двс_млн_руб_старое,
    n.двс_млн_руб_новое,
    n.двс_млн_руб_новое - COALESCE(o.двс_млн_руб_старое, 0) AS дельта_двс,

    o.накоп_млн_руб_старое,
    n.накоп_млн_руб_новое,
    n.накоп_млн_руб_новое - COALESCE(o.накоп_млн_руб_старое, 0) AS дельта_накоп
FROM new n
FULL OUTER JOIN old o
    ON n.cli_id = o.cli_id
INNER JOIN transfers t
    ON t.cli_id = COALESCE(n.cli_id, o.cli_id)
ORDER BY дельта_накоп ASC; -- можно сменить DESC, если нужно видеть рост
