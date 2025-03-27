Ниже приведены два отдельных запроса, которые:

Складывают в одну строку для каждого (WEEK_START, WEEK, BANK_NAME) входящие и исходящие суммы (учитывая, что у вас либо INCOMING_* поля заполнены, либо OUTGOING_*, либо оба).

Вычисляют SALDO_SUM – разницу между входящими и исходящими.

Ранжируют банки по этому сальдо, выбирают топ-5 + «ОСТАЛЬНЫЕ_БАНКИ».

При этом:

Первый запрос считает «сальдо входящих» как \( \text{SALDO_SUM} = \text{INCOMING_SUM} - \text{OUTGOING_SUM} \) и берёт топ-5 по убыванию (самые большие «чистые входящие»).

Второй запрос делает «обратную» разницу, \( \text{SALDO_SUM} = \text{OUTGOING_SUM} - \text{INCOMING_SUM} \), то есть топ-5 по исходящему сальдо.

Обратите внимание, что внизу у нас одна строка с BANK_NAME = 'ОСТАЛЬНЫЕ_БАНКИ', в которой суммируется сальдо всех, кто не попал в топ-5.

1. Топ-5 банков по сальдо входящих (\(\text{in} - \text{out}\))
sql
Копировать
Редактировать
WITH cte AS (
    -- 1) Собираем единое название банка,
    --    входящие и исходящие (если для какой-то строки OUTGOING_* пусто, берём 0).
    SELECT
        WEEK_START,
        WEEK,
        COALESCE(INCOMING_BANK_NAME, OUTGOING_BANK_NAME) AS BANK_NAME,
        NVL(INCOMING_SUM_TRANS, 0) AS inc_sum,
        NVL(OUTGOING_SUM_TRANS, 0) AS out_sum
    FROM sbp_statistic
),
summed AS (
    -- 2) Если в одной неделе для банка есть несколько строк,
    --    суммируем все входящие/исходящие, чтобы иметь одну запись на (WEEK_START, WEEK, BANK_NAME).
    SELECT
        WEEK_START,
        WEEK,
        BANK_NAME,
        SUM(inc_sum) AS inc_sum,
        SUM(out_sum) AS out_sum
    FROM cte
    GROUP BY WEEK_START, WEEK, BANK_NAME
),
ranked AS (
    -- 3) Считаем сальдо (in - out). Ранжируем по убыванию,
    --    чтобы найти топ-5.
    SELECT
        WEEK_START,
        WEEK,
        BANK_NAME,
        inc_sum,
        out_sum,
        (inc_sum - out_sum) AS SALDO_SUM,
        ROW_NUMBER() OVER (
            PARTITION BY WEEK_START, WEEK
            ORDER BY (inc_sum - out_sum) DESC
        ) AS rn
    FROM summed
)
-- 4) Выбираем топ-5 + "ОСТАЛЬНЫЕ_БАНКИ"
SELECT
    WEEK_START,
    WEEK,
    BANK_NAME,
    SALDO_SUM
FROM ranked
WHERE rn <= 5

UNION ALL

SELECT
    WEEK_START,
    WEEK,
    'ОСТАЛЬНЫЕ_БАНКИ' AS BANK_NAME,
    SUM(SALDO_SUM) AS SALDO_SUM
FROM ranked
WHERE rn > 5
GROUP BY WEEK_START, WEEK
ORDER BY WEEK_START, WEEK, BANK_NAME;
Итог
В каждой неделе вы получите:

До 5 строк с конкретными банками, имеющими наибольшее \(\text{inc_sum} - \text{out_sum}\).

Одну строку, где BANK_NAME = 'ОСТАЛЬНЫЕ_БАНКИ', где будет сумма сальдо «всех прочих» банков.

2. Топ-5 банков по сальдо исходящих (\(\text{out} - \text{in}\))
Чтобы сделать «обратное» сальдо (то есть у кого максимальный перевес исходящих над входящими), меняем только формулу и порядок сортировки:

sql
Копировать
Редактировать
WITH cte AS (
    SELECT
        WEEK_START,
        WEEK,
        COALESCE(INCOMING_BANK_NAME, OUTGOING_BANK_NAME) AS BANK_NAME,
        NVL(INCOMING_SUM_TRANS, 0) AS inc_sum,
        NVL(OUTGOING_SUM_TRANS, 0) AS out_sum
    FROM sbp_statistic
),
summed AS (
    SELECT
        WEEK_START,
        WEEK,
        BANK_NAME,
        SUM(inc_sum) AS inc_sum,
        SUM(out_sum) AS out_sum
    FROM cte
    GROUP BY WEEK_START, WEEK, BANK_NAME
),
ranked AS (
    -- Теперь SALDO_SUM = out_sum - inc_sum
    SELECT
        WEEK_START,
        WEEK,
        BANK_NAME,
        inc_sum,
        out_sum,
        (out_sum - inc_sum) AS SALDO_SUM,
        ROW_NUMBER() OVER (
            PARTITION BY WEEK_START, WEEK
            ORDER BY (out_sum - inc_sum) DESC
        ) AS rn
    FROM summed
)
SELECT
    WEEK_START,
    WEEK,
    BANK_NAME,
    SALDO_SUM
FROM ranked
WHERE rn <= 5

UNION ALL

SELECT
    WEEK_START,
    WEEK,
    'ОСТАЛЬНЫЕ_БАНКИ' AS BANK_NAME,
    SUM(SALDO_SUM) AS SALDO_SUM
FROM ranked
WHERE rn > 5
GROUP BY WEEK_START, WEEK
ORDER BY WEEK_START, WEEK, BANK_NAME;
В результате вы увидите топ-5 банков с наибольшим \(\text{out_sum} - \text{inc_sum}\).

Принцип работы
CTE cte: собираем одно название банка (BANK_NAME) из INCOMING_BANK_NAME или OUTGOING_BANK_NAME (через COALESCE). Если входящие / исходящие были пусты, берём 0.

summed: если в одной неделе у одного и того же банка могли быть несколько строк, суммируем их.

ranked: считаем SALDO_SUM как (in - out) или (out - in) и используем оконную функцию ROW_NUMBER(), сортируя по убыванию SALDO_SUM.

Основной SELECT:

rn <= 5: выводим банки, попавшие в топ-5.

«ОСТАЛЬНЫЕ_БАНКИ»: группируем все записи с rn > 5.

Таким образом, вы получаете именно то, что просили:

Топ-5 по сальдо входящих (положительная разница входящих над исходящими)

И аналогично топ-5 по сальдо исходящих (обратная разница).

При этом мелкие банки, которые встречаются только в одном направлении, автоматически попадут либо в топ-5 (если сальдо достаточно велико), либо в «ОСТАЛЬНЫЕ_БАНКИ».
