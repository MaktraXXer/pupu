Да. Оптимальная архитектура здесь — **одна таблица с версиями параметров + inline table-valued function**.

Она лучше обычной scalar UDF: inline TVF представляет собой один `SELECT`, который SQL Server разворачивает в общий план запроса. Scalar UDF inlining зависит от версии SQL Server и выполнения ряда условий, поэтому для массового расчёта через `CROSS APPLY` inline TVF надёжнее. ([Microsoft Learn][1])

## Полный скрипт

```sql
USE [ALM_TEST];
GO

SET NOCOUNT ON;
GO


/*==============================================================
  1. Создание схемы morgach
==============================================================*/

IF SCHEMA_ID(N'morgach') IS NULL
BEGIN
    EXEC(N'CREATE SCHEMA [morgach] AUTHORIZATION [dbo];');
END;
GO


/*==============================================================
  2. Таблица параметров CPR-модели

  FLOAT(53) соответствует double precision / numpy float64.
  Одна строка = одна версия модели.
==============================================================*/

IF OBJECT_ID(N'morgach.cpr_model_params', N'U') IS NULL
BEGIN
    CREATE TABLE morgach.cpr_model_params
    (
        model_id       smallint      NOT NULL,
        model_code     varchar(50)   NOT NULL,
        model_name     nvarchar(250) NOT NULL,
        is_active      bit           NOT NULL,

        [L]             float(53)     NOT NULL,
        r0              float(53)     NOT NULL,
        t0              float(53)     NOT NULL,

        M_gamma         float(53)     NOT NULL,
        M_rho           float(53)     NOT NULL,
        M_lam           float(53)     NOT NULL,
        M_delta         float(53)     NOT NULL,
        M_alpha         float(53)     NOT NULL,

        U_gamma         float(53)     NOT NULL,
        U_rho           float(53)     NOT NULL,
        U_lam           float(53)     NOT NULL,
        U_delta         float(53)     NOT NULL,
        U_alpha         float(53)     NOT NULL,

        k1_gamma        float(53)     NOT NULL,
        k1_rho          float(53)     NOT NULL,
        k1_lam          float(53)     NOT NULL,
        k1_delta        float(53)     NOT NULL,
        k1_alpha        float(53)     NOT NULL,

        k2_gamma        float(53)     NOT NULL,
        k2_rho          float(53)     NOT NULL,
        k2_lam          float(53)     NOT NULL,
        k2_delta        float(53)     NOT NULL,
        k2_alpha        float(53)     NOT NULL,

        I1_gamma        float(53)     NOT NULL,
        I1_rho          float(53)     NOT NULL,
        I1_lam          float(53)     NOT NULL,
        I1_delta        float(53)     NOT NULL,
        I1_alpha        float(53)     NOT NULL,
        I1_c            float(53)     NOT NULL,

        I2_gamma        float(53)     NOT NULL,
        I2_rho          float(53)     NOT NULL,
        I2_lam          float(53)     NOT NULL,
        I2_delta        float(53)     NOT NULL,
        I2_alpha        float(53)     NOT NULL,
        I2_c            float(53)     NOT NULL,

        created_at      datetime2(0)  NOT NULL
            CONSTRAINT DF_cpr_model_params_created_at
            DEFAULT SYSUTCDATETIME(),

        updated_at      datetime2(0)  NOT NULL
            CONSTRAINT DF_cpr_model_params_updated_at
            DEFAULT SYSUTCDATETIME(),

        CONSTRAINT PK_cpr_model_params
            PRIMARY KEY CLUSTERED (model_id),

        CONSTRAINT UQ_cpr_model_params_code
            UNIQUE (model_code)
    );
END;
GO


/*==============================================================
  3. Загрузка параметров Python-модели

  Скрипт можно запускать повторно:
  существующая модель обновится, отсутствующая — добавится.
==============================================================*/

UPDATE morgach.cpr_model_params
SET
    model_code = 'IA_BANK_ALL',
    model_name = N'CPR — модель на данных ИА и Банка',
    is_active  = 1,

    [L] = 3.038193496337776E-18,
    r0  = 8.9,
    t0  = 34.0,

    M_gamma = 0.04472468830742795,
    M_rho   = 0.8673608976519765,
    M_lam   = 0.03341378652276081,
    M_delta = 6.948999158160149,
    M_alpha = 0.18469518793237027,

    U_gamma = 0.08155042033445704,
    U_rho   = 0.8743247668084375,
    U_lam   = 0.015119701860476701,
    U_delta = 4.942452201409712,
    U_alpha = 0.03579769051047221,

    k1_gamma = 0.01762747827267916,
    k1_rho   = 2.2981038040093793,
    k1_lam   = 0.09916038563776206,
    k1_delta = 2.9250629261163885,
    k1_alpha = 1.0E-9,

    k2_gamma = 0.2200080629363351,
    k2_rho   = 6.082770991469117,
    k2_lam   = 0.055867605849304725,
    k2_delta = 2.8227018067036416,
    k2_alpha = 0.48370017098736806,

    I1_gamma = 2.367058751435384,
    I1_rho   = 0.340384777617535,
    I1_lam   = 0.8059008235361652,
    I1_delta = 2.775710951472079,
    I1_alpha = -0.24141376245568005,
    I1_c     = -9.887386576344273,

    I2_gamma = 0.6869658833212233,
    I2_rho   = 1.3129219471590028,
    I2_lam   = 6.233758336095655E-6,
    I2_delta = 1.4807940032454379,
    I2_alpha = -1.0000000001201425E-9,
    I2_c     = -0.0010513672328571013,

    updated_at = SYSUTCDATETIME()
WHERE model_id = 1;


IF @@ROWCOUNT = 0
BEGIN
    INSERT INTO morgach.cpr_model_params
    (
        model_id,
        model_code,
        model_name,
        is_active,

        [L],
        r0,
        t0,

        M_gamma,
        M_rho,
        M_lam,
        M_delta,
        M_alpha,

        U_gamma,
        U_rho,
        U_lam,
        U_delta,
        U_alpha,

        k1_gamma,
        k1_rho,
        k1_lam,
        k1_delta,
        k1_alpha,

        k2_gamma,
        k2_rho,
        k2_lam,
        k2_delta,
        k2_alpha,

        I1_gamma,
        I1_rho,
        I1_lam,
        I1_delta,
        I1_alpha,
        I1_c,

        I2_gamma,
        I2_rho,
        I2_lam,
        I2_delta,
        I2_alpha,
        I2_c
    )
    VALUES
    (
        1,
        'IA_BANK_ALL',
        N'CPR — модель на данных ИА и Банка',
        1,

        3.038193496337776E-18,
        8.9,
        34.0,

        0.04472468830742795,
        0.8673608976519765,
        0.03341378652276081,
        6.948999158160149,
        0.18469518793237027,

        0.08155042033445704,
        0.8743247668084375,
        0.015119701860476701,
        4.942452201409712,
        0.03579769051047221,

        0.01762747827267916,
        2.2981038040093793,
        0.09916038563776206,
        2.9250629261163885,
        1.0E-9,

        0.2200080629363351,
        6.082770991469117,
        0.055867605849304725,
        2.8227018067036416,
        0.48370017098736806,

        2.367058751435384,
        0.340384777617535,
        0.8059008235361652,
        2.775710951472079,
        -0.24141376245568005,
        -9.887386576344273,

        0.6869658833212233,
        1.3129219471590028,
        6.233758336095655E-6,
        1.4807940032454379,
        -1.0000000001201425E-9,
        -0.0010513672328571013
    );
END;
GO


/*==============================================================
  4. Функция расчёта CPR

  Вход:
      @model_id  — версия модели
      @incentive — стимул, процентные пункты
      @age       — выдержка, месяцы
      @rate      — ставка кредита, % годовых

  Выход:
      cpr — CPR в долях единицы

  Например:
      0.2663 = 26.63%
==============================================================*/

CREATE OR ALTER FUNCTION morgach.fn_predict_cpr
(
    @model_id  smallint,
    @incentive float(53),
    @age       float(53),
    @rate      float(53)
)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
(
    SELECT
        cpr =
              p.[L]
            + (component_values.M_val - p.[L])
                * final_sigmoid.sig1
            + (component_values.U_val - component_values.M_val)
                * final_sigmoid.sig2

    FROM morgach.cpr_model_params AS p

    /* rate - r0 */
    CROSS APPLY
    (
        VALUES
        (
            @rate - p.r0
        )
    ) AS rate_difference(R)

    /* Аргументы sigmoid для влияния ставки */
    CROSS APPLY
    (
        VALUES
        (
            p.M_alpha  * rate_difference.R,
            p.U_alpha  * rate_difference.R,
            p.k1_alpha * rate_difference.R,
            p.k2_alpha * rate_difference.R,
            p.I1_alpha * rate_difference.R,
            p.I2_alpha * rate_difference.R
        )
    ) AS rate_arguments
    (
        M_x,
        U_x,
        k1_x,
        k2_x,
        I1_x,
        I2_x
    )

    /*
      Устойчивый расчёт expit.

      Вместо EXP(-x) напрямую используется EXP(-ABS(x)).
      Поэтому при больших отрицательных аргументах не возникает
      переполнения EXP().
    */
    CROSS APPLY
    (
        VALUES
        (
            EXP(-ABS(rate_arguments.M_x)),
            EXP(-ABS(rate_arguments.U_x)),
            EXP(-ABS(rate_arguments.k1_x)),
            EXP(-ABS(rate_arguments.k2_x)),
            EXP(-ABS(rate_arguments.I1_x)),
            EXP(-ABS(rate_arguments.I2_x))
        )
    ) AS rate_exp
    (
        M_e,
        U_e,
        k1_e,
        k2_e,
        I1_e,
        I2_e
    )

    CROSS APPLY
    (
        VALUES
        (
            1.0 + p.M_delta *
                CASE
                    WHEN rate_arguments.M_x >= 0.0
                        THEN 1.0 / (1.0 + rate_exp.M_e)
                    ELSE rate_exp.M_e / (1.0 + rate_exp.M_e)
                END,

            1.0 + p.U_delta *
                CASE
                    WHEN rate_arguments.U_x >= 0.0
                        THEN 1.0 / (1.0 + rate_exp.U_e)
                    ELSE rate_exp.U_e / (1.0 + rate_exp.U_e)
                END,

            1.0 + p.k1_delta *
                CASE
                    WHEN rate_arguments.k1_x >= 0.0
                        THEN 1.0 / (1.0 + rate_exp.k1_e)
                    ELSE rate_exp.k1_e / (1.0 + rate_exp.k1_e)
                END,

            1.0 + p.k2_delta *
                CASE
                    WHEN rate_arguments.k2_x >= 0.0
                        THEN 1.0 / (1.0 + rate_exp.k2_e)
                    ELSE rate_exp.k2_e / (1.0 + rate_exp.k2_e)
                END,

            1.0 + p.I1_delta *
                CASE
                    WHEN rate_arguments.I1_x >= 0.0
                        THEN 1.0 / (1.0 + rate_exp.I1_e)
                    ELSE rate_exp.I1_e / (1.0 + rate_exp.I1_e)
                END,

            1.0 + p.I2_delta *
                CASE
                    WHEN rate_arguments.I2_x >= 0.0
                        THEN 1.0 / (1.0 + rate_exp.I2_e)
                    ELSE rate_exp.I2_e / (1.0 + rate_exp.I2_e)
                END
        )
    ) AS rate_factors
    (
        M_rate_factor,
        U_rate_factor,
        k1_rate_factor,
        k2_rate_factor,
        I1_rate_factor,
        I2_rate_factor
    )

    /* Возрастные множители */
    CROSS APPLY
    (
        VALUES
        (
            1.0 + p.M_rho
                * EXP(-p.M_lam * @age),

            1.0 + p.U_rho
                * EXP(-p.U_lam * @age),

            1.0 + p.k1_rho
                * EXP(-p.k1_lam * @age),

            1.0 + p.k2_rho
                * EXP(-p.k2_lam * @age),

            1.0 + p.I1_rho
                * LOG(1.0 + p.I1_lam * @age),

            1.0 + p.I2_rho
                * LOG(1.0 + p.I2_lam * @age)
        )
    ) AS age_factors
    (
        M_age_factor,
        U_age_factor,
        k1_age_factor,
        k2_age_factor,
        I1_age_factor,
        I2_age_factor
    )

    /* Значения M, U, k1, k2, I1, I2 */
    CROSS APPLY
    (
        VALUES
        (
            p.M_gamma
                * age_factors.M_age_factor
                * rate_factors.M_rate_factor,

            p.U_gamma
                * age_factors.U_age_factor
                * rate_factors.U_rate_factor,

            p.k1_gamma
                * age_factors.k1_age_factor
                * rate_factors.k1_rate_factor,

            p.k2_gamma
                * age_factors.k2_age_factor
                * rate_factors.k2_rate_factor,

            p.I1_gamma
                * age_factors.I1_age_factor
                * rate_factors.I1_rate_factor
                + p.I1_c,

            p.I2_gamma
                * age_factors.I2_age_factor
                * rate_factors.I2_rate_factor
                + p.I2_c
        )
    ) AS component_values
    (
        M_val,
        U_val,
        k1_val,
        k2_val,
        I1_val,
        I2_val
    )

    /* Аргументы двух итоговых sigmoid */
    CROSS APPLY
    (
        VALUES
        (
            component_values.k1_val
                * (@incentive - component_values.I1_val),

            component_values.k2_val
                * (@incentive - component_values.I2_val)
        )
    ) AS final_arguments
    (
        sig1_x,
        sig2_x
    )

    CROSS APPLY
    (
        VALUES
        (
            EXP(-ABS(final_arguments.sig1_x)),
            EXP(-ABS(final_arguments.sig2_x))
        )
    ) AS final_exp
    (
        sig1_e,
        sig2_e
    )

    CROSS APPLY
    (
        VALUES
        (
            CASE
                WHEN final_arguments.sig1_x >= 0.0
                    THEN 1.0 / (1.0 + final_exp.sig1_e)
                ELSE final_exp.sig1_e
                    / (1.0 + final_exp.sig1_e)
            END,

            CASE
                WHEN final_arguments.sig2_x >= 0.0
                    THEN 1.0 / (1.0 + final_exp.sig2_e)
                ELSE final_exp.sig2_e
                    / (1.0 + final_exp.sig2_e)
            END
        )
    ) AS final_sigmoid
    (
        sig1,
        sig2
    )

    WHERE p.model_id = @model_id
);
GO
```

## Проверка против Python

Для параметров:

```python
incentive = -1
age = 12
rate = 17.3
```

Python выдаёт:

[
CPR=0.26633241091071364
]

или:

[
26.633241091071364%
]

Проверка в SQL:

```sql
SELECT
    f.cpr AS sql_cpr,
    100.0 * f.cpr AS sql_cpr_pct,

    CAST(0.26633241091071364 AS float) AS python_cpr,

    ABS
    (
        f.cpr
        - CAST(0.26633241091071364 AS float)
    ) AS absolute_difference

FROM morgach.fn_predict_cpr
(
    1,      -- model_id
    -1.0,   -- incentive
    12.0,   -- age
    17.3    -- rate
) AS f;
```

Ожидаемый результат:

|           sql_cpr |      sql_cpr_pct |
| ----------------: | ---------------: |
| 0.266332410910714 | 26.6332410910714 |

Расхождение может быть только на последних знаках `float`, поскольку и NumPy, и `FLOAT(53)` используют двойную точность. Математические функции SQL Server `EXP` и `LOG` также работают с `float`. ([Microsoft Learn][2])

## Массовый расчёт по таблице

Допустим, есть таблица:

```sql
dbo.mortgage_forecast
```

со столбцами:

```text
con_id
payment_period
incentive
age
rate
```

Тогда:

```sql
SELECT
    t.con_id,
    t.payment_period,
    t.incentive,
    t.age,
    t.rate,

    f.cpr,
    100.0 * f.cpr AS cpr_pct

FROM dbo.mortgage_forecast AS t

CROSS APPLY morgach.fn_predict_cpr
(
    1,
    t.incentive,
    t.age,
    t.rate
) AS f;
```

Если CPR нужен для дальнейшего расчёта SMM:

```sql
SELECT
    t.*,

    f.cpr,

    1.0 - POWER
    (
        1.0 - f.cpr,
        1.0 / 12.0
    ) AS smm

FROM dbo.mortgage_forecast AS t

CROSS APPLY morgach.fn_predict_cpr
(
    1,
    t.incentive,
    t.age,
    t.rate
) AS f;
```

Если досрочный платёж считается от OD на начало месяца:

```sql
SELECT
    t.*,

    f.cpr,

    smm_calc.smm,

    t.od_begin * smm_calc.smm
        AS modeled_prepayment

FROM dbo.mortgage_forecast AS t

CROSS APPLY morgach.fn_predict_cpr
(
    1,
    t.incentive,
    t.age,
    t.rate
) AS f

CROSS APPLY
(
    VALUES
    (
        1.0 - POWER
        (
            1.0 - f.cpr,
            1.0 / 12.0
        )
    )
) AS smm_calc(smm);
```

## Важные соответствия Python

* `expit()` реализован устойчиво, без риска переполнения `EXP()` при большом отрицательном аргументе.
* `np.exp()` зеркально заменён на `EXP()`.
* `np.log1p(x)` заменён на `LOG(1+x)`.
* Все расчёты выполняются в `FLOAT(53)`.
* Результат возвращается в долях единицы.
* Параметр `t0` сохранён в таблице, но **не используется**, потому что он не используется в исходной функции `predict_cpr()`.
* Никаких ограничений или обрезки CPR диапазоном (0\ldots1) не добавлено, поскольку их нет в Python.
* SQL-аналог broadcasting — обычный расчёт функции для каждой строки через `CROSS APPLY`.

Схема создаётся стандартным `CREATE SCHEMA`, а `SCHEMABINDING` защищает таблицу параметров от изменений структуры, которые могли бы незаметно сломать функцию. ([Microsoft Learn][3])

[1]: https://learn.microsoft.com/et-ee/sql/t-sql/statements/create-function-transact-sql?view=sql-server-linux-ver17&utm_source=chatgpt.com "CREATE FUNCTION (Transact-SQL) - SQL Server | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/sql/t-sql/functions/exp-transact-sql?view=sql-server-ver17&utm_source=chatgpt.com "EXP (Transact-SQL) - SQL Server | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-schema-transact-sql?view=sql-server-ver17&utm_source=chatgpt.com "CREATE SCHEMA (Transact-SQL) - SQL Server | Microsoft Learn"
