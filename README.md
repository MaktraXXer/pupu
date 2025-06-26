USE ALM_TEST;
GO
/*====================================================================
   work.prc_test_newUL_cube
   ► новые депозиты ЮЛ (Крупн. + ССВ)
   ► TYPE = 'Начало' (i = 2)
   ► оставляет rolled-up категории ‘all …’ через CUBE
====================================================================*/
CREATE OR ALTER PROCEDURE work.prc_test_newUL_cube
        @dt_Start date = NULL,
        @dt_End   date = NULL
AS
BEGIN
    SET NOCOUNT ON;

    /*----------------------------------------------------------------
      1. Диапазон дат
    ----------------------------------------------------------------*/
    SET @dt_Start = ISNULL(@dt_Start,'2024-06-01');
    SET @dt_End   = ISNULL(@dt_End ,DATEADD(day,-2,CAST(GETDATE() AS date)));

    DROP TABLE IF EXISTS #calendar;
    SELECT  [Date]
    INTO    #calendar
    FROM    ALM.info.VW_Calendar WITH (NOLOCK)
    WHERE   [Date] BETWEEN @dt_Start AND @dt_End;

    /*----------------------------------------------------------------
      2. #depSpreads  – базовые строки (Date, TYPE, CLI_SUBTYPE и пр.)
         здесь уже считаем весовые показатели
    ----------------------------------------------------------------*/
    DROP TABLE IF EXISTS #depSpreads;
    SELECT
        dep.DT_OPEN                              AS [Date],       -- i = 2
        'Начало'                                 AS [TYPE],
        dep.CLI_SUBTYPE,
        ISNULL(dep.MARGIN_TYPE,'Прочий_тип')     AS MARGIN_TYPE,
        term.TERM_GROUP,
        CAST(dep.IS_OPTION AS varchar(255))      AS IS_OPTION,
        ISNULL(dep.SEG_NAME,'Прочий_сегмент')    AS SEG_NAME,

        SUM(saldo.OUT_RUB)                                                  AS BALANCE_RUB,
        SUM(saldo.OUT_RUB*dep.MATUR)                 / SUM(saldo.OUT_RUB)   AS MATUR,
        SUM(saldo.OUT_RUB*dep.LIQ_ФОР)               / SUM(saldo.OUT_RUB)   AS ФОР,
        SUM(saldo.OUT_RUB*dep.LIQ_ССВ_Fcast)         / SUM(saldo.OUT_RUB)   AS ССВ,
        SUM(saldo.OUT_RUB*dep.ALM_OptionRate)        / SUM(saldo.OUT_RUB)   AS ALM_OptionRate,
        SUM(saldo.OUT_RUB*
            CASE WHEN dep.MATUR<=365
                 THEN dep.MonthlyCONV_RoisFix
                 ELSE dep.MonthlyCONV_KBD END)      / SUM(saldo.OUT_RUB)   AS MonthlyCONV_OIS,
        SUM(saldo.OUT_RUB*
            CASE WHEN dep.CLI_SUBTYPE<>'ФЛ'
                 THEN dep.MonthlyCONV_LIQ_TransfertRate
                 ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                             dep.MonthlyCONV_LIQ_TransfertRate)
            END)                                     / SUM(saldo.OUT_RUB)  AS MonthlyCONV_TransfertRate,
        SUM(saldo.OUT_RUB*
            CASE WHEN dep.CLI_SUBTYPE<>'ФЛ' THEN
                   CASE WHEN dep.MonthlyCONV_LIQ_TransfertRate-
                              (dep.MonthlyCONV_Rate+dep.LIQ_ССВ_Fcast) >=0
                        THEN dep.MonthlyCONV_LIQ_TransfertRate
                        ELSE dep.MonthlyCONV_Rate+dep.LIQ_ССВ_Fcast
                   END
                 ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                             dep.MonthlyCONV_LIQ_TransfertRate)
            END)                                     / SUM(saldo.OUT_RUB)  AS MonthlyCONV_TransfertRate_MOD,
        SUM(saldo.OUT_RUB*dep.MonthlyCONV_ALM_TransfertRate) / SUM(saldo.OUT_RUB) AS MonthlyCONV_ALM_TransfertRate,
        SUM(saldo.OUT_RUB*dep.MonthlyCONV_KBD)               / SUM(saldo.OUT_RUB) AS MonthlyCONV_KBD,
        SUM(saldo.OUT_RUB*ISNULL(dep.MonthlyCONV_ForecastKeyRate,0))
                                                           / SUM(saldo.OUT_RUB) AS MonthlyCONV_ForecastKeyRate,
        SUM(saldo.OUT_RUB*dep.MonthlyCONV_Rate)             / SUM(saldo.OUT_RUB) AS MonthlyCONV_Rate
    INTO  #depSpreads
    FROM  LIQUIDITY.liq.DepositInterestsRate   dep   WITH (NOLOCK)
    JOIN  #calendar                            cal   ON cal.[Date]=dep.DT_OPEN      -- i = 2
    JOIN  LIQUIDITY.liq.DepositContract_Saldo  saldo WITH (NOLOCK)
          ON  saldo.CON_ID = dep.CON_ID
          AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
    LEFT JOIN LIQUIDITY.liq.man_TermGroup      term  WITH (NOLOCK)
          ON  dep.MATUR BETWEEN term.TERM_FROM AND term.TERM_TO
    WHERE dep.DT_REP = (SELECT MAX(DT_REP) FROM LIQUIDITY.liq.DepositInterestsRate)
      AND dep.CUR = 'RUR'
      AND dep.CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
      AND dep.IsDomRF = 0
      AND dep.RATE > 0.01
      AND ISNULL(dep.isfloat,0)=0
      AND dep.LIQ_ФОР IS NOT NULL
      AND dep.MonthlyCONV_RATE IS NOT NULL
      AND dep.MonthlyCONV_RATE BETWEEN dep.MonthlyCONV_ForecastKeyRate-0.07
                                   AND dep.MonthlyCONV_ForecastKeyRate+0.07
      AND saldo.OUT_RUB <> 0
      AND dep.CLI_ID <> 3731800
      AND CASE WHEN dep.CLI_SUBTYPE<>'ФЛ'
               THEN dep.MonthlyCONV_LIQ_TransfertRate
               ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                           dep.MonthlyCONV_LIQ_TransfertRate)
          END IS NOT NULL
    GROUP BY
         dep.DT_OPEN, dep.CLI_SUBTYPE,
         ISNULL(dep.MARGIN_TYPE,'Прочий_тип'),
         term.TERM_GROUP,
         dep.IS_OPTION,
         ISNULL(dep.SEG_NAME,'Прочий_сегмент');

    /*----------------------------------------------------------------
      3. #cube – CUBE по тем же измерениям
         (никаких UNION, чтобы не задвоить баланс)
    ----------------------------------------------------------------*/
    DROP TABLE IF EXISTS #cube;
    SELECT
        CAST([Date] AS smalldatetime)                 AS [Date],
        [TYPE],
        CLI_SUBTYPE,
        MARGIN_TYPE,
        TERM_GROUP,
        IS_OPTION,
        SEG_NAME,

        SUM(BALANCE_RUB)                                              AS BALANCE_RUB,
        SUM(BALANCE_RUB*МАТUR)                 / SUM(BALANCE_RUB)     AS MATUR,
        SUM(BALANCE_RUB*ФОР)                   / SUM(BALANCE_RUB)     AS ФОР,
        SUM(BALANCE_RUB*ССВ)                   / SUM(BALANCE_RUB)     AS ССВ,
        SUM(BALANCE_RUB*ALM_OptionRate)        / SUM(BALANCE_RUB)     AS ALM_OptionRate,
        SUM(BALANCE_RUB*MonthlyCONV_OIS)       / SUM(BALANCE_RUB)     AS MonthlyCONV_OIS,
        SUM(BALANCE_RUB*MonthlyCONV_TransfertRate) /
             SUM(BALANCE_RUB)                                      AS MonthlyCONV_TransfertRate,
        SUM(BALANCE_RUB*MonthlyCONV_TransfertRate_MOD) /
             SUM(BALANCE_RUB)                                      AS MonthlyCONV_TransfertRate_MOD,
        SUM(BALANCE_RUB*MonthlyCONV_ALM_TransfertRate) /
             SUM(BALANCE_RUB)                                      AS MonthlyCONV_ALM_TransfertRate,
        SUM(BALANCE_RUB*MonthlyCONV_KBD) / SUM(BALANCE_RUB)         AS MonthlyCONV_KBD,
        SUM(BALANCE_RUB*MonthlyCONV_ForecastKeyRate) /
             SUM(BALANCE_RUB)                                      AS MonthlyCONV_ForecastKeyRate,
        SUM(BALANCE_RUB*MonthlyCONV_Rate) / SUM(BALANCE_RUB)        AS MonthlyCONV_Rate
    INTO #cube
    FROM #depSpreads
    GROUP BY CUBE
        ([Date], [TYPE], CLI_SUBTYPE, MARGIN_TYPE,
         TERM_GROUP, IS_OPTION, SEG_NAME);

    /*----------------------------------------------------------------
      4. Нормализуем NULL → 'all …',  CLI_SUBTYPE NULL → 'ЮЛ'
    ----------------------------------------------------------------*/
    UPDATE #cube
    SET  TERM_GROUP = ISNULL(TERM_GROUP,'all termgroup'),
         IS_OPTION  = ISNULL(IS_OPTION ,'all option_type'),
         SEG_NAME   = ISNULL(SEG_NAME  ,'all segment'),
         MARGIN_TYPE= ISNULL(MARGIN_TYPE,'all margin'),
         CLI_SUBTYPE= ISNULL(CLI_SUBTYPE,'ЮЛ');      -- агрегатный уровень


    /*----------------------------------------------------------------
      5. Грузим в витрину
    ----------------------------------------------------------------*/
    IF OBJECT_ID('ALM_TEST.WORK.test_newUL_cube') IS NULL
    BEGIN
        SELECT *, CAST(NULL AS datetime) AS DT_INSERT
        INTO   ALM_TEST.WORK.test_newUL_cube
        FROM   #cube
        WHERE  1=0;                              -- создаём структуру
    END

    DELETE FROM ALM_TEST.WORK.test_newUL_cube
    WHERE [Date] BETWEEN @dt_Start AND @dt_End;

    INSERT INTO ALM_TEST.WORK.test_newUL_cube
    SELECT *, GETDATE()
    FROM   #cube;
END
GO
