Оставляю шаг стимула 0,1 п.п., как и для ставки.

Диапазоны:

* стимул: от −30 до +30 п.п. с шагом 0,1 п.п.;
* выдержка: от 0 до 380 месяцев;
* ставка: от 0% до 39% с шагом 0,1 п.п.

Итого получится 89 531 571 строка.

Создание таблицы

USE [ALM_TEST];
GO
IF OBJECT_ID(N'morgach.cpr_lookup', N'U') IS NULL
BEGIN
    CREATE TABLE morgach.cpr_lookup
    (
        model_id smallint NOT NULL,
        /*
            Стимул в долях:
            -0.300 = -30 п.п.
            -0.002 = -0.2 п.п.
             0.300 = +30 п.п.
        */
        incentive decimal(6,3) NOT NULL,
        /*
            Выдержка в месяцах:
            от 0 до 380.
        */
        age_months smallint NOT NULL,
        /*
            Ставка в долях:
            0.000 = 0%
            0.100 = 10%
            0.390 = 39%
        */
        rate decimal(5,3) NOT NULL,
        /*
            CPR и SMM в долях:
            0.15 = 15%.
        */
        cpr float(53) NOT NULL,
        smm float(53) NOT NULL,
        calculated_at datetime2(0) NOT NULL
            CONSTRAINT DF_cpr_lookup_calculated_at
            DEFAULT SYSUTCDATETIME(),
        CONSTRAINT PK_cpr_lookup
            PRIMARY KEY CLUSTERED
            (
                model_id,
                incentive,
                age_months,
                rate
            ),
        CONSTRAINT CK_cpr_lookup_incentive
            CHECK (incentive BETWEEN -0.300 AND 0.300),
        CONSTRAINT CK_cpr_lookup_age
            CHECK (age_months BETWEEN 0 AND 380),
        CONSTRAINT CK_cpr_lookup_rate
            CHECK (rate BETWEEN 0.000 AND 0.390)
    );
END;
GO

Заполнение таблицы

Стимул и ставка строятся через целые числа, а потом делятся на 1000. Это надёжнее, чем многократно прибавлять 0.001, потому что не возникает накопления погрешности float.

USE [ALM_TEST];
GO
SET NOCOUNT ON;
SET XACT_ABORT ON;
GO
DECLARE
    @model_id smallint,
    @age smallint = 0;
/*
    Максимальная активная версия модели.
*/
SELECT
    @model_id = MAX(model_id)
FROM morgach.cpr_model_params
WHERE is_active = 1;
IF @model_id IS NULL
BEGIN
    THROW 50001,
          N'В morgach.cpr_model_params отсутствует активная CPR-модель.',
          1;
END;
/*
    Удаляем старый расчёт этой версии модели.
*/
DELETE FROM morgach.cpr_lookup
WHERE model_id = @model_id;
/*
    На каждом возрасте рассчитывается:
    601 стимул:
    -0.300 ... +0.300 с шагом 0.001
    391 ставка:
     0.000 ... 0.390 с шагом 0.001
    На один возраст:
    601 * 391 = 235  - тысяч строк.
*/
WHILE @age <= 380
BEGIN
    ;WITH nums AS
    (
        SELECT TOP (1000)
            n = CONVERT
            (
                int,
                ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1
            )
        FROM sys.all_objects AS a
        CROSS JOIN sys.all_objects AS b
    ),
    incentives AS
    (
        SELECT
            incentive = CAST
            (
                (n - 300) / 1000.0
                AS decimal(6,3)
            )
        FROM nums
        WHERE n BETWEEN 0 AND 600
    ),
    rates AS
    (
        SELECT
            rate = CAST
            (
                n / 1000.0
                AS decimal(5,3)
            )
        FROM nums
        WHERE n BETWEEN 0 AND 390
    )
    INSERT INTO morgach.cpr_lookup
    (
        model_id,
        incentive,
        age_months,
        rate,
        cpr,
        smm
    )
    SELECT
        cpr_result.model_id,
        i.incentive,
        @age,
        r.rate,
        cpr_result.cpr,
        1.0
        - POWER
        (
            1.0 - cpr_result.cpr,
            1.0 / 12.0
        ) AS smm
    FROM incentives AS i
    CROSS JOIN rates AS r
    CROSS APPLY morgach.fn_predict_cpr
    (
        CONVERT(float(53), i.incentive),
        CONVERT(float(53), @age),
        CONVERT(float(53), r.rate)
    ) AS cpr_result;
    PRINT
    (
        N'Рассчитана выдержка '
        + CONVERT(nvarchar(10), @age)
        + N' мес.'
    );
    SET @age += 1;
END;
GO

Проверка результата

SELECT
    model_id,
    COUNT_BIG(*) AS row_count,
    MIN(incentive) AS min_incentive,
    MAX(incentive) AS max_incentive,
    MIN(age_months) AS min_age_months,
    MAX(age_months) AS max_age_months,
    MIN(rate) AS min_rate,
    MAX(rate) AS max_rate,
    MIN(cpr) AS min_cpr,
    MAX(cpr) AS max_cpr
FROM morgach.cpr_lookup
GROUP BY model_id;

Ожидаемое количество строк для одной модели:

89531571

Проверка уникальности комбинаций

SELECT
    COUNT_BIG(*) AS total_rows,
    COUNT_BIG
    (
        DISTINCT CONCAT
        (
            model_id, '|',
            incentive, '|',
            age_months, '|',
            rate
        )
    ) AS unique_rows
FROM morgach.cpr_lookup;

Хотя отдельная проверка не обязательна: первичный ключ уже запрещает дублирование комбинации.

JOIN к справочнику

Если данные коллег округлены до шага 0,1 п.п.:

SELECT
    t.con_id,
    t.payment_period,
    t.incentive,
    t.age,
    t.rate,
    l.model_id,
    l.cpr,
    l.smm
FROM dbo.mortgage_forecast AS t
CROSS APPLY
(
    VALUES
    (
        CAST
        (
            ROUND(t.incentive, 3)
            AS decimal(6,3)
        ),
        CAST
        (
            ROUND(t.rate, 3)
            AS decimal(5,3)
        ),
        CAST
        (
            ROUND(t.age, 0)
            AS smallint
        )
    )
) AS j
(
    incentive_join,
    rate_join,
    age_join
)
JOIN morgach.cpr_lookup AS l
  ON l.model_id =
     (
         SELECT MAX(model_id)
         FROM morgach.cpr_model_params
         WHERE is_active = 1
     )
 AND l.incentive = j.incentive_join
 AND l.age_months = j.age_join
 AND l.rate = j.rate_join;

Для быстрого поиска по последней модели текущий первичный ключ подходит, поскольку model_id стоит первым.
