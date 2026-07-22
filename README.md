USE [ALM_TEST];
GO
CREATE OR ALTER FUNCTION morgach.fn_predict_cpr_scalar
(
    @incentive float(53),  -- -0.002 = стимул -0.2 п.п.
    @age       float(53),  -- выдержка в месяцах
    @rate      float(53)   -- 0.10 = ставка 10% годовых
)
RETURNS float(53)
WITH SCHEMABINDING
AS
BEGIN

    DECLARE @cpr float(53);

    SELECT
        @cpr =
              p.[L]
            + (cv.M_val - p.[L]) * fs.sig1
            + (cv.U_val - cv.M_val) * fs.sig2
    FROM morgach.cpr_model_params AS p

    /* Перевод входных величин в формат как в питоне
       0.10->10.0%
    -0.002->-0.2 п.п.
    */
    CROSS APPLY
    (
        VALUES
        (
            @rate * 100.0,
            @incentive * 100.0
        )
    ) AS mi
    (
        rate_pct,
        incentive_pp
    )

    /* R = rate - r0 */
    CROSS APPLY
    (
        VALUES
        (
            mi.rate_pct - p.r0
        )
    ) AS rd(R)

    /* Аргументы rate-sigmoid */
    CROSS APPLY
    (
        VALUES
        (
            p.M_alpha  * rd.R,
            p.U_alpha  * rd.R,
            p.k1_alpha * rd.R,
            p.k2_alpha * rd.R,
            p.I1_alpha * rd.R,
            p.I2_alpha * rd.R
        )
    ) AS ra
    (
        M_x,
        U_x,
        k1_x,
        k2_x,
        I1_x,
        I2_x
    )

    /*rate_factor = 1 + delta * expit(alpha * R)
       EXP(-ABS(x)) для устойчивого expit функции
       https://stackoverflow.com/questions/51976461/optimal-way-of-defining-a-numerically-stable-sigmoid-function-for-a-list-in-pyth
       */
    CROSS APPLY
    (
        VALUES
        (
            EXP(-ABS(ra.M_x)),
            EXP(-ABS(ra.U_x)),
            EXP(-ABS(ra.k1_x)),
            EXP(-ABS(ra.k2_x)),
            EXP(-ABS(ra.I1_x)),
            EXP(-ABS(ra.I2_x))
        )
    ) AS re
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
                    WHEN ra.M_x >= 0.0
                        THEN 1.0 / (1.0 + re.M_e)
                    ELSE re.M_e / (1.0 + re.M_e)
                END,

            1.0 + p.U_delta *
                CASE
                    WHEN ra.U_x >= 0.0
                        THEN 1.0 / (1.0 + re.U_e)
                    ELSE re.U_e / (1.0 + re.U_e)
                END,

            1.0 + p.k1_delta *
                CASE
                    WHEN ra.k1_x >= 0.0
                        THEN 1.0 / (1.0 + re.k1_e)
                    ELSE re.k1_e / (1.0 + re.k1_e)
                END,

            1.0 + p.k2_delta *
                CASE
                    WHEN ra.k2_x >= 0.0
                        THEN 1.0 / (1.0 + re.k2_e)
                    ELSE re.k2_e / (1.0 + re.k2_e)
                END,

            1.0 + p.I1_delta *
                CASE
                    WHEN ra.I1_x >= 0.0
                        THEN 1.0 / (1.0 + re.I1_e)
                    ELSE re.I1_e / (1.0 + re.I1_e)
                END,

            1.0 + p.I2_delta *
                CASE
                    WHEN ra.I2_x >= 0.0
                        THEN 1.0 / (1.0 + re.I2_e)
                    ELSE re.I2_e / (1.0 + re.I2_e)
                END
        )
    ) AS rf
    (
        M_rate_factor,
        U_rate_factor,
        k1_rate_factor,
        k2_rate_factor,
        I1_rate_factor,
        I2_rate_factor
    )

    /*Возрастные множители*/
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
    ) AS af
    (
        M_age_factor,
        U_age_factor,
        k1_age_factor,
        k2_age_factor,
        I1_age_factor,
        I2_age_factor
    )

    /* Компоненты M, U, k1, k2, I1, I2 */
    CROSS APPLY
    (
        VALUES
        (
            p.M_gamma
                * af.M_age_factor
                * rf.M_rate_factor,

            p.U_gamma
                * af.U_age_factor
                * rf.U_rate_factor,

            p.k1_gamma
                * af.k1_age_factor
                * rf.k1_rate_factor,

            p.k2_gamma
                * af.k2_age_factor
                * rf.k2_rate_factor,

            p.I1_gamma
                * af.I1_age_factor
                * rf.I1_rate_factor
                + p.I1_c,

            p.I2_gamma
                * af.I2_age_factor
                * rf.I2_rate_factor
                + p.I2_c
        )
    ) AS cv
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
            cv.k1_val * (mi.incentive_pp - cv.I1_val),
            cv.k2_val * (mi.incentive_pp - cv.I2_val)
        )
    ) AS fa
    (
        sig1_x,
        sig2_x
    )

    CROSS APPLY
    (
        VALUES
        (
            EXP(-ABS(fa.sig1_x)),
            EXP(-ABS(fa.sig2_x))
        )
    ) AS fe
    (
        sig1_e,
        sig2_e
    )

    CROSS APPLY
    (
        VALUES
        (
            CASE
                WHEN fa.sig1_x >= 0.0
                    THEN 1.0 / (1.0 + fe.sig1_e)
                ELSE fe.sig1_e / (1.0 + fe.sig1_e)
            END,

            CASE
                WHEN fa.sig2_x >= 0.0
                    THEN 1.0 / (1.0 + fe.sig2_e)
                ELSE fe.sig2_e / (1.0 + fe.sig2_e)
            END
        )
    ) AS fs
    (
        sig1,
        sig2
    )

    /* WHERE обязательно после всех CROSS APPLY */
    WHERE
        p.is_active = 1
        AND p.model_id =
        (
            SELECT MAX(pm.model_id)
            FROM morgach.cpr_model_params AS pm
            WHERE pm.is_active = 1
        );

    RETURN @cpr;

END;
GO
