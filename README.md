Я бы начал с гипотезы, что 5 млрд рублей не обязательно являются реальным оттоком вкладов — возможно, часть портфеля выпадает из твоих расчётных выборок.

В присланных SQL вижу несколько конкретных причин, почему статистики могут не сходиться.

### 1. Что уже не так в сопоставлении

| Проблема                                         | Почему важно                                                                                                                                 |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Разные даты                                      | Общий остаток смотришь с 29.10 по 02.11, действующие вклады и новые привлечения — с 30.09 по 02.11, поклиентную динамику — с 01.11 по 02.11. |
| Разные фильтры                                   | В общем остатке нет `acc_role = 'LIAB'`, а в расчётах действующих и новых вкладов он есть.                                                   |
| Разные исключения продуктов                      | Во втором скрипте написано `НадёжныйVIP` вместо `Надёжный VIP`, а `Надёжныйпроцент` обрабатывается неодинаково.                              |
| Неполный охват закрытий                          | Плановые погашения не включают досрочно закрытые вклады, а в расчёте действующих вкладов остаются только договоры с `dt_close_plan > 02.11`. |
| Новые привлечения считаются по конечному остатку | Скрипт берёт открытые с 01.10 вклады, существующие на 02.11. Это не весь фактический объём открытий за период.                               |
| Поклиентная выборка ограничена                   | `OFFSET 0 ROWS FETCH NEXT 2500 ROWS ONLY` показывает лишь 2500 клиентов, поэтому совокупный отток может быть распределён между остальными.   |

Эти расхождения видны непосредственно в условиях твоих запросов.&#x20;

Вставленный текст(5).txt

Вставленный текст(5).txt

Вставленный текст(5).txt



### 2. Как я предлагаю искать 5 млрд

Вместо того чтобы пытаться объяснить разницу сразу тремя статистиками, сначала нужно построить точный мост остатков на уровне `con_id`:

\\[ \Delta ОСЗ = \underbrace{\text{Появившиеся вклады}}\_{\text{не было → есть}} - \underbrace{\text{Исчезнувшие вклады}}\_{\text{были → нет}} + \underbrace{\text{Изменение действующих}}\_{\text{были → остались}} \\]

Для двух снимков с одинаковыми фильтрами это точное балансовое равенство, независимо от плановых дат закрытия.

Далее можно определить, какие именно договоры обеспечили снижение на 5 млрд, отделить плановые закрытия от досрочных, а также проверить, не изменились ли продукт, сегмент или классификация договора.

### 3. SQL для поиска конкретных вкладов

Скрипт использует точно те же фильтры, что твой первый запрос, но сравнивает два среза по `con_id`. Он покажет суммы исчезнувших, появившихся и продолжающих действовать вкладов.

```

USE ALM;
SET NOCOUNT ON;

DECLARE @DateOne date = '2026-10-29';
DECLARE @DateTwo date = '2026-11-02';

DROP TABLE IF EXISTS #snap;
DROP TABLE IF EXISTS #bridge;

/* 1. Исходные остатки — фильтры первого скрипта */

SELECT
    dt_rep,
    con_id,
    cli_id,
    TSEGMENTNAME,
    PROD_NAME_res,
    acc_role,
    CAST(dt_open AS date) AS dt_open,
    CAST(dt_close_plan AS date) AS dt_close_plan,
    CAST(out_rub AS decimal(28,2)) AS out_rub
INTO #snap
FROM ALM.ALM.balance_rest_all
WHERE dt_rep IN (@DateOne, @DateTwo)
  AND section_name = N'Срочные'
  AND block_name = N'Привлечение ФЛ'
  AND od_flag = 1
  AND cur = '810'
  AND out_rub >= 0
  AND PROD_NAME_res NOT IN (
      N'Надёжный прайм', N'Надёжный VIP',
      N'Надёжный премиум', N'Надёжный промо',
      N'Надёжный старт', N'Надёжный Т2',
      N'Надёжный Мегафон', N'Надёжный процент',
      N'Могучий', N'Надёжный',
      N'ДОМа надёжно', N'Всё в ДОМ'
  );

/* 2. Проверка двух исходных срезов */

SELECT
    dt_rep,
    SUM(out_rub) / 1e9 AS balance_bn,
    COUNT(DISTINCT con_id) AS contracts,
    SUM(CASE WHEN con_id IS NULL
             THEN out_rub ELSE 0 END) / 1e9
        AS no_con_id_bn,
    SUM(CASE WHEN acc_role <> N'LIAB'
                  OR acc_role IS NULL
             THEN out_rub ELSE 0 END) / 1e9
        AS non_liab_bn
FROM #snap
GROUP BY dt_rep
ORDER BY dt_rep;

/* 3. Сопоставление договоров */

;WITH agg AS (
    SELECT
        dt_rep,
        con_id,
        MAX(TSEGMENTNAME) AS segment,
        MAX(PROD_NAME_res) AS product,
        MAX(dt_open) AS dt_open,
        MAX(dt_close_plan) AS dt_close_plan,
        SUM(out_rub) AS balance
    FROM #snap
    WHERE con_id IS NOT NULL
    GROUP BY dt_rep, con_id
)
SELECT
    COALESCE(a.con_id, b.con_id) AS con_id,
    COALESCE(a.segment, b.segment) AS segment,
    COALESCE(a.product, b.product) AS product,
    a.dt_close_plan AS planned_close,
    b.dt_open AS open_date,

    ISNULL(a.balance, 0) AS balance_before,
    ISNULL(b.balance, 0) AS balance_after,
    ISNULL(b.balance, 0) -
        ISNULL(a.balance, 0) AS delta,

    CASE
        WHEN a.con_id IS NULL
             AND b.dt_open > @DateOne
            THEN N'1. Новые вклады'

        WHEN a.con_id IS NULL
            THEN N'2. Появились, но открыты ранее'

        WHEN b.con_id IS NULL
             AND a.dt_close_plan > @DateOne
             AND a.dt_close_plan <= @DateTwo
            THEN N'3. Исчезли, плановый срок в периоде'

        WHEN b.con_id IS NULL
            THEN N'4. Исчезли вне планового срока'

        ELSE N'5. Действующие вклады'
    END AS category

INTO #bridge
FROM (SELECT * FROM agg WHERE dt_rep = @DateOne) a
FULL OUTER JOIN
     (SELECT * FROM agg WHERE dt_rep = @DateTwo) b
    ON a.con_id = b.con_id;

/* 4. Балансовый мост по причинам изменения */

SELECT
    category,
    COUNT(*) AS contracts,
    SUM(balance_before) / 1e9 AS before_bn,
    SUM(balance_after) / 1e9 AS after_bn,
    SUM(delta) / 1e9 AS delta_bn
FROM #bridge
GROUP BY category
ORDER BY category;

/* 5. Какие продукты и сегменты дали изменение */

SELECT
    product,
    segment,
    COUNT(*) AS contracts,
    SUM(delta) / 1e9 AS delta_bn
FROM #bridge
GROUP BY product, segment
ORDER BY ABS(SUM(delta)) DESC;

/* 6. ТОП-100 договоров по абсолютному изменению */

SELECT TOP (100)
    con_id,
    category,
    segment,
    product,
    planned_close,
    open_date,
    balance_before,
    balance_after,
    delta
FROM #bridge
ORDER BY ABS(delta) DESC;

/* 7. Где теперь находятся исчезнувшие договоры */

;WITH lost AS (
    SELECT TOP (100) *
    FROM #bridge
    WHERE category LIKE N'3.%'
       OR category LIKE N'4.%'
    ORDER BY balance_before DESC
)
SELECT
    l.con_id,
    l.balance_before,
    l.planned_close,
    t.section_name,
    t.PROD_NAME_res,
    t.block_name,
    t.acc_role,
    t.od_flag,
    t.cur,
    t.out_rub
FROM lost l
LEFT JOIN ALM.ALM.balance_rest_all t
    ON t.con_id = l.con_id
   AND t.dt_rep = @DateTwo
ORDER BY l.balance_before DESC;

```

В последнем результате важно смотреть, присутствуют ли исчезнувшие договоры в таблице вообще:

- Если договор исчез из выбранного портфеля, но существует с другим `section_name`, `PROD_NAME_res` или `acc_role`, причина может быть в переклассификации.
- Если договор отсутствует полностью, нужно проверять его фактическое закрытие и историю остатков.
- Если сумма по `no_con_id_bn` существенна, такие записи нужно разобрать отдельно: мост по `con_id` их не учитывает.

### 4. Ещё одна потенциально серьёзная ошибка

В скрипте новых привлечений у тебя есть соединение с `ALM_TEST.WORK.ForecastKey_Cache` по дате открытия и сроку вклада.&#x20;

Вставленный текст(5).txt



Если в этой таблице несколько строк на одну комбинацию даты и срока, `LEFT JOIN` размножает записи вкладов, завышая статистику новых привлечений.

Проверить можно так:

```
SELECT
    DT_REP,
    TERM,
    COUNT(*) AS cnt
FROM ALM_TEST.WORK.ForecastKey_Cache
WHERE DT_REP BETWEEN '2026-10-01' AND '2026-11-02'
GROUP BY DT_REP, TERM
HAVING COUNT(*) > 1;
```

Если запрос что-то вернёт, нужно проверить, не завышается ли сумма `#new_bal`.

### 5. Что проверил бы в первую очередь

Сначала — результат № 4 (балансовый мост). Он покажет, в какой категории находится отрицательная дельта, и позволит понять, насколько она совпадает с твоими расчётами.

Затем — результат № 5: продукты и сегменты, сформировавшие отток. И только потом разбирал бы отдельные договоры из результата № 6.

Наконец, я бы повторил контроль для 02.11–03.11: если 5 млрд исчезли 2 ноября и вернулись 3 ноября, стоит прежде всего проверять полноту загрузки и корректность витрины, а не депозитное поведение клиентов.

Главная задача: сначала получить точную раскладку изменения ОСЗ на уровне договоров, затем сопоставить её с твоими тремя расчётами. Так мы сможем найти не просто разницу в 5 млрд, а конкретные договоры и условие SQL, из-за которого они не попали в статистику.
