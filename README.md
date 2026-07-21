Ниже функция, которая принимает:

* @rate = 0.10 как ставку 10% годовых;
* @incentive = -0.002 как стимул −0,2 п.п.;
* @age — выдержку в месяцах.

Внутри функции ставка и стимул умножаются на 100, поскольку исходная Python-модель использует значения 10.0 и -0.2.

Модель автоматически выбирается как активная модель с максимальным model_id.

USE [ALM_TEST];
GO
CREATE OR ALTER FUNCTION morgach.fn_predict_cpr
(
    @incentive float(53),  -- -0.002 = -0.2 п.п.
    @age       float(53),  -- выдержка в месяцах
    @rate      float(53)   -- 0.10 = 10% годовых
)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
(
    SELECT
        model_id = p.model_id,
        /*
            CPR возвращается в долях единицы:
            0.15 = 15% CPR
        */
        cpr =
              p.[L]
            + (component_values.M_val - p.[L])
                * final_sigmoid.sig1
            + (component_values.U_val - component_values.M_val)
                * final_sigmoid.sig2
    FROM morgach.cpr_model_params AS p
    /*
        Автоматически выбираем последнюю активную
        версию модели.
    */
    WHERE
        p.is_active = 1
        AND p.model_id =
        (
            SELECT MAX(p_max.model_id)
            FROM morgach.cpr_model_params AS p_max
            WHERE p_max.is_active = 1
        )
    /*
        Внешние единицы:
            rate      = 0.10
            incentive = -0.002
        Единицы Python-модели:
            rate      = 10.0
            incentive = -0.2
    */
    CROSS APPLY
    (
        VALUES
        (
            @rate * 100.0,
            @incentive * 100.0
        )
    ) AS model_inputs
    (
        rate_pct,
        incentive_pp
    )
    /*
        Отклонение ставки кредита от r0.
        Python:
            R = rate - r0
    */
    CROSS APPLY
    (
        VALUES
        (
            model_inputs.rate_pct - p.r0
        )
    ) AS rate_difference(R)
    /*
        Аргументы sigmoid для влияния ставки.
    */
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
        Устойчивый аналог scipy.special.expit().
        Вместо EXP(-x) используется EXP(-ABS(x)),
        чтобы не получить переполнение при больших
        отрицательных аргументах.
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
            1.0
            + p.M_delta
            * CASE
                  WHEN rate_arguments.M_x >= 0.0
                      THEN 1.0 / (1.0 + rate_exp.M_e)
                  ELSE rate_exp.M_e / (1.0 + rate_exp.M_e)
              END,
            1.0
            + p.U_delta
            * CASE
                  WHEN rate_arguments.U_x >= 0.0
                      THEN 1.0 / (1.0 + rate_exp.U_e)
                  ELSE rate_exp.U_e / (1.0 + rate_exp.U_e)
              END,
            1.0
            + p.k1_delta
            * CASE
                  WHEN rate_arguments.k1_x >= 0.0
                      THEN 1.0 / (1.0 + rate_exp.k1_e)
                  ELSE rate_exp.k1_e / (1.0 + rate_exp.k1_e)
              END,
            1.0
            + p.k2_delta
            * CASE
                  WHEN rate_arguments.k2_x >= 0.0
                      THEN 1.0 / (1.0 + rate_exp.k2_e)
                  ELSE rate_exp.k2_e / (1.0 + rate_exp.k2_e)
              END,
            1.0
            + p.I1_delta
            * CASE
                  WHEN rate_arguments.I1_x >= 0.0
                      THEN 1.0 / (1.0 + rate_exp.I1_e)
                  ELSE rate_exp.I1_e / (1.0 + rate_exp.I1_e)
              END,
            1.0
            + p.I2_delta
            * CASE
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
    /*
        Возрастные множители.
        Для M, U, k1, k2:
            1 + rho * EXP(-lambda * age)
        Для I1, I2:
            1 + rho * LOG(1 + lambda * age)
    */
    CROSS APPLY
    (
        VALUES
        (
            1.0
            + p.M_rho
            * EXP(-p.M_lam * @age),
            1.0
            + p.U_rho
            * EXP(-p.U_lam * @age),
            1.0
            + p.k1_rho
            * EXP(-p.k1_lam * @age),
            1.0
            + p.k2_rho
            * EXP(-p.k2_lam * @age),
            1.0
            + p.I1_rho
            * LOG(1.0 + p.I1_lam * @age),
            1.0
            + p.I2_rho
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
    /*
        Значения шести компонент модели.
    */
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
    /*
        Аргументы двух итоговых sigmoid.
    */
    CROSS APPLY
    (
        VALUES
        (
            component_values.k1_val
            * (
                  model_inputs.incentive_pp
                - component_values.I1_val
              ),
            component_values.k2_val
            * (
                  model_inputs.incentive_pp
                - component_values.I2_val
              )
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
);
GO

Вызов для одного значения

Исходный пример Python:

incentive = -1
age = 12
rate = 17.3

Теперь передаётся так:

SELECT
    f.model_id,
    f.cpr,
    100.0 * f.cpr AS cpr_pct
FROM morgach.fn_predict_cpr
(
    -0.010,  -- стимул -1 п.п.
    12.0,    -- выдержка 12 месяцев
    0.173    -- ставка 17.3%
) AS f;

Ожидаемый CPR:

0.2663324109

или:

26.63324109%

Пример со стимулом −0,2 п.п. и ставкой 10%

SELECT
    f.model_id,
    f.cpr,
    100.0 * f.cpr AS cpr_pct
FROM morgach.fn_predict_cpr
(
    -0.002,  -- стимул -0.2 п.п.
    12.0,    -- выдержка 12 месяцев
    0.10     -- ставка 10%
) AS f;

Массовый вызов по таблице

Допустим, в таблице:

* rate хранится как 0.173;
* incentive хранится как -0.010;
* age хранится в месяцах.

SELECT
    t.con_id,
    t.payment_period,
    t.incentive,
    t.age,
    t.rate,
    f.model_id,
    f.cpr,
    100.0 * f.cpr AS cpr_pct
FROM dbo.mortgage_forecast AS t
CROSS APPLY morgach.fn_predict_cpr
(
    t.incentive,
    t.age,
    t.rate
) AS f;

CPR, SMM и модельный досрочный платёж

Если досрочный платёж применяется к OD на начало месяца:

SELECT
    t.con_id,
    t.payment_period,
    t.od_begin,
    t.incentive,
    t.age,
    t.rate,
    f.model_id,
    f.cpr,
    smm_calc.smm,
    t.od_begin * smm_calc.smm
        AS modeled_prepayment
FROM dbo.mortgage_forecast AS t
CROSS APPLY morgach.fn_predict_cpr
(
    t.incentive,
    t.age,
    t.rate
) AS f
CROSS APPLY
(
    VALUES
    (
        1.0
        - POWER
          (
              1.0 - f.cpr,
              1.0 / 12.0
          )
    )
) AS smm_calc(smm);

Проверка последней используемой модели

SELECT
    model_id,
    model_code,
    model_name,
    is_active
FROM morgach.cpr_model_params
WHERE model_id =
(
    SELECT MAX(model_id)
    FROM morgach.cpr_model_params
    WHERE is_active = 1
);

Если активных моделей нет, функция вернёт ноль строк. Поэтому при добавлении новой версии необходимо устанавливать:

is_active = 1

Максимальный model_id среди активных моделей будет использован автоматически.
