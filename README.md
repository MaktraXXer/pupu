USE [ALM_TEST];
GO

IF OBJECT_ID(N'morgach.cpr_lookup', N'U') IS NULL
BEGIN
    CREATE TABLE morgach.cpr_lookup
    (
        model_id       smallint      NOT NULL,
        incentive      decimal(6,3)  NOT NULL,
        age_months     smallint      NOT NULL,
        rate           decimal(5,3)  NOT NULL,
        cpr            float(53)     NOT NULL,
        smm            float(53)     NOT NULL,
        calculated_at  datetime2(0)  NOT NULL
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


USE [ALM_TEST];
GO

SET NOCOUNT ON;
SET XACT_ABORT ON;
GO

DECLARE
    @model_id   smallint,
    @age_months smallint = 0;

SELECT
    @model_id = MAX(model_id)
FROM morgach.cpr_model_params
WHERE is_active = 1;

IF @model_id IS NULL
BEGIN
    THROW 50001, N'Отсутствует активная CPR-модель.', 1;
END;


CREATE TABLE #incentives
(
    incentive decimal(6,3) NOT NULL
        PRIMARY KEY
);

;WITH numbers AS
(
    SELECT -300 AS n

    UNION ALL

    SELECT n + 1
    FROM numbers
    WHERE n < 300
)
INSERT INTO #incentives
(
    incentive
)
SELECT
    CAST(n / 1000.0 AS decimal(6,3))
FROM numbers
OPTION (MAXRECURSION 0);


CREATE TABLE #rates
(
    rate decimal(5,3) NOT NULL
        PRIMARY KEY
);

;WITH numbers AS
(
    SELECT 0 AS n

    UNION ALL

    SELECT n + 1
    FROM numbers
    WHERE n < 390
)
INSERT INTO #rates
(
    rate
)
SELECT
    CAST(n / 1000.0 AS decimal(5,3))
FROM numbers
OPTION (MAXRECURSION 0);


DELETE FROM morgach.cpr_lookup
WHERE model_id = @model_id;


WHILE @age_months <= 380
BEGIN
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
        c.model_id,
        i.incentive,
        @age_months,
        r.rate,
        c.cpr,
        1.0 - POWER(1.0 - c.cpr, 1.0 / 12.0)

    FROM #incentives AS i

    CROSS JOIN #rates AS r

    CROSS APPLY morgach.fn_predict_cpr
    (
        CONVERT(float(53), i.incentive),
        CONVERT(float(53), @age_months),
        CONVERT(float(53), r.rate)
    ) AS c;

    SET @age_months += 1;
END;


DROP TABLE #incentives;
DROP TABLE #rates;
GO
