USE ALM;
GO

SET NOCOUNT ON;

DECLARE @json NVARCHAR(MAX) = N'
{
  "M": {"gamma": 0.0360939164692716, "rho": 1.0377775602882766, "lambda": 0.024857536772070893, "delta": 6.925237520382742, "alpha": 0.22226022918575383},
  "U": {"gamma": 0.061474955843779176, "rho": 1.1150420397596015, "lambda": 0.008899299449885328, "delta": 4.943916673321318, "alpha": 0.0528430517312853},
  "k1": {"gamma": 0.014283610454898324, "rho": 2.2802397579127414, "lambda": 0.04335350678674854, "delta": 2.924946608611938, "alpha": 1e-08},
  "k2": {"gamma": 0.17330087748525627, "rho": 6.0908259310548996, "lambda": 0.030179240111745678, "delta": 2.8058966829731418, "alpha": 0.5894963465876183},
  "I1": {"gamma": 2.4144684876370874, "rho": 0.09918583228190554, "lambda": 0.8614593343035482, "delta": 2.8469056035661753, "alpha": -0.2652372622899474, "c": -9.91480922788922},
  "I2": {"gamma": 0.6493917452883262, "rho": 1.3130029424958412, "lambda": 1e-08, "delta": 1.4723792072053574, "alpha": -1.0000000000076169e-08, "c": -0.022378865558275493},
  "L": 0.0,
  "r0": 8.9,
  "t0": 35.0
}';

DECLARE @now DATETIME2 = SYSDATETIME();

BEGIN TRY
    BEGIN TRAN;

    UPDATE p
    SET
        -- Метаданные модели
        p.model_code = 'model_params_20260901',
        p.model_name = N'CPR — модель из файла model_params_20260901.json',
        p.created_at = @now,
        p.updated_at = @now,

        -- Общие параметры
        p.L = j.L,
        p.r0 = j.r0,
        p.t0 = j.t0,

        -- M
        p.M_gamma = j.M_gamma,
        p.M_rho = j.M_rho,
        p.M_lam = j.M_lam,
        p.M_delta = j.M_delta,
        p.M_alpha = j.M_alpha,

        -- U
        p.U_gamma = j.U_gamma,
        p.U_rho = j.U_rho,
        p.U_lam = j.U_lam,
        p.U_delta = j.U_delta,
        p.U_alpha = j.U_alpha,

        -- k1
        p.k1_gamma = j.k1_gamma,
        p.k1_rho = j.k1_rho,
        p.k1_lam = j.k1_lam,
        p.k1_delta = j.k1_delta,
        p.k1_alpha = j.k1_alpha,

        -- k2
        p.k2_gamma = j.k2_gamma,
        p.k2_rho = j.k2_rho,
        p.k2_lam = j.k2_lam,
        p.k2_delta = j.k2_delta,
        p.k2_alpha = j.k2_alpha,

        -- I1
        p.I1_gamma = j.I1_gamma,
        p.I1_rho = j.I1_rho,
        p.I1_lam = j.I1_lam,
        p.I1_delta = j.I1_delta,
        p.I1_alpha = j.I1_alpha,
        p.I1_c = j.I1_c,

        -- I2
        p.I2_gamma = j.I2_gamma,
        p.I2_rho = j.I2_rho,
        p.I2_lam = j.I2_lam,
        p.I2_delta = j.I2_delta,
        p.I2_alpha = j.I2_alpha,
        p.I2_c = j.I2_c

    FROM dbo.cpr_model_params p
    CROSS APPLY OPENJSON(@json)
    WITH (
        L FLOAT '$.L',
        r0 FLOAT '$.r0',
        t0 FLOAT '$.t0',

        M_gamma FLOAT '$.M.gamma',
        M_rho FLOAT '$.M.rho',
        M_lam FLOAT '$.M.lambda',
        M_delta FLOAT '$.M.delta',
        M_alpha FLOAT '$.M.alpha',

        U_gamma FLOAT '$.U.gamma',
        U_rho FLOAT '$.U.rho',
        U_lam FLOAT '$.U.lambda',
        U_delta FLOAT '$.U.delta',
        U_alpha FLOAT '$.U.alpha',

        k1_gamma FLOAT '$.k1.gamma',
        k1_rho FLOAT '$.k1.rho',
        k1_lam FLOAT '$.k1.lambda',
        k1_delta FLOAT '$.k1.delta',
        k1_alpha FLOAT '$.k1.alpha',

        k2_gamma FLOAT '$.k2.gamma',
        k2_rho FLOAT '$.k2.rho',
        k2_lam FLOAT '$.k2.lambda',
        k2_delta FLOAT '$.k2.delta',
        k2_alpha FLOAT '$.k2.alpha',

        I1_gamma FLOAT '$.I1.gamma',
        I1_rho FLOAT '$.I1.rho',
        I1_lam FLOAT '$.I1.lambda',
        I1_delta FLOAT '$.I1.delta',
        I1_alpha FLOAT '$.I1.alpha',
        I1_c FLOAT '$.I1.c',

        I2_gamma FLOAT '$.I2.gamma',
        I2_rho FLOAT '$.I2.rho',
        I2_lam FLOAT '$.I2.lambda',
        I2_delta FLOAT '$.I2.delta',
        I2_alpha FLOAT '$.I2.alpha',
        I2_c FLOAT '$.I2.c'
    ) j
    WHERE p.model_id = 1;

    IF @@ROWCOUNT <> 1
        THROW 50001, 'Model update failed: expected exactly one row.', 1;

    COMMIT;

    SELECT *
    FROM dbo.cpr_model_params
    WHERE model_id = 1;

END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK;

    THROW;
END CATCH;
