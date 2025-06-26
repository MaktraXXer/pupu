### Готовая процедура `work.prc_test_UL_cube_full`

\*- «Начало» (новые сделки, i = 2)
\*- «Срез»   (портфель на дату, i = 1)
\*- только **ЮЛ Крупн. / ЮЛ ССВ**
\*- все требуемые «измерения» — `SEG_NAME, TERM_GROUP, IS_OPTION, MARGIN_TYPE, IS_PDR, IS_FINANCE_LCR`
\*- расчёт `MARGIN_TYPE (PLUS/MINUS)` и `UL_OUTFLOW_LCR` 1-в-1 c логикой коллег
\*- CUBE-агрегация → строки «all …» (баланс бьётся с пос-сделочной выборкой)

```sql
USE ALM_TEST;
GO
/*====================================================================
   work.prc_test_UL_cube_full
   ► ЮЛ (Крупн. + ССВ)            ► TYPE = 'Начало'  и  'Срез'
   ► все нужные фильтры + CUBE    ► баланс совпадает с деталями
====================================================================*/
CREATE OR ALTER PROCEDURE work.prc_test_UL_cube_full
        @dt_Start date = NULL,
        @dt_End   date = NULL
AS
BEGIN
    SET NOCOUNT ON;

    /*--------- 1. Диапазон дат (по-умолчанию –16…-2 дня) -----------*/
    SET @dt_Start = ISNULL(@dt_Start, DATEADD(day,-16,CAST(GETDATE() AS date)));
    SET @dt_End   = ISNULL(@dt_End  , DATEADD(day, -2,CAST(GETDATE() AS date)));

    /*--------- 2. Календарь ---------------------------------------*/
    DROP TABLE IF EXISTS #calendar;
    SELECT  [Date]
    INTO    #calendar
    FROM    ALM.info.VW_Calendar WITH (NOLOCK)
    WHERE   [Date] BETWEEN @dt_Start AND @dt_End;

    /*================================================================
      3.  Пос-сделочная выборка + нужные «точки» времени
    =================================================================*/
    DROP TABLE IF EXISTS #raw;
    ------------------------------------------------  «Срез»  (i = 1)
    SELECT  cal.[Date],                     'Срез'   AS [TYPE],
            dep.*,
            saldo.OUT_RUB
    INTO    #raw
    FROM    LIQUIDITY.liq.DepositInterestsRate dep WITH (NOLOCK)
    JOIN    #calendar                cal   ON dep.DT_OPEN <= cal.[Date]
                                          AND dep.DT_CLOSE >  cal.[Date]
    JOIN    LIQUIDITY.liq.DepositContract_Saldo saldo WITH (NOLOCK)
               ON  saldo.CON_ID = dep.CON_ID
               AND cal.[Date]  BETWEEN saldo.DT_FROM AND saldo.DT_TO
    WHERE   dep.DT_REP = (SELECT MAX(DT_REP)
                          FROM LIQUIDITY.liq.DepositInterestsRate)
      AND   dep.CUR              = 'RUR'
      AND   dep.CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
      AND   dep.IsDomRF          = 0
      AND   dep.RATE             > 0.01
      AND   ISNULL(dep.isfloat,0)=0
      AND   dep.LIQ_ФОР IS NOT NULL
      AND   dep.MonthlyCONV_RATE IS NOT NULL
      AND   dep.MonthlyCONV_RATE BETWEEN dep.MonthlyCONV_ForecastKeyRate-0.07
                                    AND dep.MonthlyCONV_ForecastKeyRate+0.07
      AND   saldo.OUT_RUB <> 0
      AND   dep.CLI_ID    <> 3731800
      AND   /* not-null transfer */
            CASE WHEN dep.CLI_SUBTYPE<>'ФЛ'
                 THEN dep.MonthlyCONV_LIQ_TransfertRate
                 ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                             dep.MonthlyCONV_LIQ_TransfertRate)
            END IS NOT NULL

    UNION ALL                                    -- «Начало» (i = 2)
    SELECT  dep.DT_OPEN                AS [Date],'Начало',
            dep.*,
            saldo.OUT_RUB
    FROM    LIQUIDITY.liq.DepositInterestsRate dep WITH (NOLOCK)
    JOIN    #calendar                cal   ON cal.[Date] = dep.DT_OPEN
    JOIN    LIQUIDITY.liq.DepositContract_Saldo saldo WITH (NOLOCK)
               ON saldo.CON_ID = dep.CON_ID
               AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
    WHERE   dep.DT_REP = (SELECT MAX(DT_REP)
                          FROM LIQUIDITY.liq.DepositInterestsRate)
      AND   dep.CUR              = 'RUR'
      AND   dep.CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
      AND   dep.IsDomRF          = 0
      AND   dep.RATE             > 0.01
      AND   ISNULL(dep.isfloat,0)=0
      AND   dep.LIQ_ФОР IS NOT NULL
      AND   dep.MonthlyCONV_RATE IS NOT NULL
      AND   dep.MonthlyCONV_RATE BETWEEN dep.MonthlyCONV_ForecastKeyRate-0.07
                                    AND dep.MonthlyCONV_ForecastKeyRate+0.07
      AND   saldo.OUT_RUB <> 0
      AND   dep.CLI_ID    <> 3731800
      AND   CASE WHEN dep.CLI_SUBTYPE<>'ФЛ'
                 THEN dep.MonthlyCONV_LIQ_TransfertRate
                 ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                             dep.MonthlyCONV_LIQ_TransfertRate)
            END IS NOT NULL;

    /*--------- 4. Добавляем расчётные поля ------------------------*/
    ALTER TABLE #raw
        ADD MARGIN_TYPE AS
            CASE WHEN (CASE WHEN CLI_SUBTYPE<>'ФЛ'
                             THEN MonthlyCONV_LIQ_TransfertRate
                             ELSE ISNULL(MonthlyCONV_ALM_TransfertRate,
                                         MonthlyCONV_LIQ_TransfertRate)
                       END) -
                       (MonthlyCONV_Rate + LIQ_ССВ_Fcast) < 0
                 THEN 'MINUS' ELSE 'PLUS' END PERSISTED,
            UL_OUTFLOW_LCR AS                      -- формула из коллег
            CASE WHEN (CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ') AND ISNULL(IS_PDR,0)=1)
                      THEN 0.4
                 WHEN (CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ') AND ISNULL(IS_PDR,0)=0)
                      THEN 0.4 * 70.0 / (0.5*(70+MATUR+ABS(MATUR-70)))
                 ELSE 0 END PERSISTED;

    /*--------- 5. Агрегируем на полный набор измерений ------------*/
    DROP TABLE IF EXISTS #depSpreads;
    SELECT
        [Date],[TYPE],CLI_SUBTYPE,MARGIN_TYPE,
        term.TERM_GROUP,
        CAST(IS_OPTION AS varchar(255))            AS IS_OPTION,
        ISNULL(SEG_NAME,'Прочий_сегмент')          AS SEG_NAME,
        CAST(ISNULL(IS_PDR,0) AS varchar(255))     AS IS_PDR,
        CAST(ISNULL(IS_FINANCE_LCR,0) AS varchar(255)) AS IS_FINANCE_LCR,

        SUM(OUT_RUB)                                                   AS BALANCE_RUB,
        SUM(OUT_RUB*MATUR)                     / SUM(OUT_RUB)          AS MATUR,
        SUM(OUT_RUB*LIQ_ФОР)                   / SUM(OUT_RUB)          AS ФОР,
        SUM(OUT_RUB*LIQ_ССВ_Fcast)             / SUM(OUT_RUB)          AS ССВ,
        SUM(OUT_RUB*ALM_OptionRate)            / SUM(OUT_RUB)          AS ALM_OptionRate,
        SUM(OUT_RUB*
            CASE WHEN MATUR<=365 THEN MonthlyCONV_RoisFix
                                  ELSE MonthlyCONV_KBD END)
                                           / SUM(OUT_RUB)              AS MonthlyCONV_OIS,
        SUM(OUT_RUB*
            CASE WHEN CLI_SUBTYPE<>'ФЛ'
                 THEN MonthlyCONV_LIQ_TransfertRate
                 ELSE ISNULL(MonthlyCONV_ALM_TransfertRate,
                             MonthlyCONV_LIQ_TransfertRate)
            END)                                    / SUM(OUT_RUB)     AS MonthlyCONV_TransfertRate,
        SUM(OUT_RUB*
            CASE WHEN CLI_SUBTYPE<>'ФЛ' THEN
                   CASE WHEN MonthlyCONV_LIQ_TransfertRate -
                              (MonthlyCONV_Rate+LIQ_ССВ_Fcast) >= 0
                        THEN MonthlyCONV_LIQ_TransfertRate
                        ELSE MonthlyCONV_Rate+LIQ_ССВ_Fcast
                   END
                 ELSE ISNULL(MonthlyCONV_ALM_TransfertRate,
                             MonthlyCONV_LIQ_TransfertRate)
            END)                                    / SUM(OUT_RUB)     AS MonthlyCONV_TransfertRate_MOD,
        SUM(OUT_RUB*MonthlyCONV_ALM_TransfertRate)/SUM(OUT_RUB)        AS MonthlyCONV_ALM_TransfertRate,
        SUM(OUT_RUB*MonthlyCONV_KBD)              /SUM(OUT_RUB)        AS MonthlyCONV_KBD,
        SUM(OUT_RUB*ISNULL(MonthlyCONV_ForecastKeyRate,0))
                                                  /SUM(OUT_RUB)        AS MonthlyCONV_ForecastKeyRate,
        SUM(OUT_RUB*MonthlyCONV_Rate)             /SUM(OUT_RUB)        AS MonthlyCONV_Rate,
        SUM(OUT_RUB*UL_OUTFLOW_LCR)               /SUM(OUT_RUB)        AS UL_OUTFLOW_LCR
    INTO #depSpreads
    FROM #raw  r
    LEFT JOIN LIQUIDITY.liq.man_TermGroup term WITH (NOLOCK)
           ON r.MATUR BETWEEN term.TERM_FROM AND term.TERM_TO
    GROUP BY [Date],[TYPE],CLI_SUBTYPE,MARGIN_TYPE,
             term.TERM_GROUP,IS_OPTION,SEG_NAME,
             IS_PDR,IS_FINANCE_LCR;

    /*--------- 6. CUBE для «all …» уровней ------------------------*/
    DROP TABLE IF EXISTS #cube;
    SELECT
        CAST([Date] AS smalldatetime)             AS [Date],
        [TYPE], CLI_SUBTYPE, MARGIN_TYPE,
        TERM_GROUP, IS_OPTION, SEG_NAME,
        IS_PDR, IS_FINANCE_LCR,

        SUM(BALANCE_RUB)                                           AS BALANCE_RUB,
        SUM(BALANCE_RUB*MATUR)                  /SUM(BALANCE_RUB)  AS MATUR,
        SUM(BALANCE_RUB*ФОР)                    /SUM(BALANCE_RUB)  AS ФОР,
        SUM(BALANCE_RUB*ССВ)                    /SUM(BALANCE_RUB)  AS ССВ,
        SUM(BALANCE_RUB*ALM_OptionRate)         /SUM(BALANCE_RUB)  AS ALM_OptionRate,
        SUM(BALANCE_RUB*MonthlyCONV_OIS)        /SUM(BALANCE_RUB)  AS MonthlyCONV_OIS,
        SUM(BALANCE_RUB*MonthlyCONV_TransfertRate)       /SUM(BALANCE_RUB) AS MonthlyCONV_TransfertRate,
        SUM(BALANCE_RUB*MonthlyCONV_TransfertRate_MOD)   /SUM(BALANCE_RUB) AS MonthlyCONV_TransfertRate_MOD,
        SUM(BALANCE_RUB*MonthlyCONV_ALM_TransfertRate)   /SUM(BALANCE_RUB) AS MonthlyCONV_ALM_TransfertRate,
        SUM(BALANCE_RUB*MonthlyCONV_KBD)        /SUM(BALANCE_RUB)  AS MonthlyCONV_KBD,
        SUM(BALANCE_RUB*MonthlyCONV_ForecastKeyRate)     /SUM(BALANCE_RUB) AS MonthlyCONV_ForecastKeyRate,
        SUM(BALANCE_RUB*MonthlyCONV_Rate)       /SUM(BALANCE_RUB)  AS MonthlyCONV_Rate,
        SUM(BALANCE_RUB*UL_OUTFLOW_LCR)         /SUM(BALANCE_RUB)  AS UL_OUTFLOW_LCR
    INTO #cube
    FROM #depSpreads
    GROUP BY CUBE ([Date],[TYPE],CLI_SUBTYPE,MARGIN_TYPE,
                   TERM_GROUP,IS_OPTION,SEG_NAME,
                   IS_PDR,IS_FINANCE_LCR);

    /*--------- 7. NULL → 'all …' и агрегатный CLI_SUBTYPE = 'ЮЛ' --*/
    UPDATE #cube
    SET TERM_GROUP  = ISNULL(TERM_GROUP ,'all termgroup'),
        IS_OPTION   = ISNULL(IS_OPTION  ,'all option_type'),
        SEG_NAME    = ISNULL(SEG_NAME   ,'all segment'),
        MARGIN_TYPE = ISNULL(MARGIN_TYPE,'all margin'),
        IS_PDR      = ISNULL(IS_PDR     ,'all PDR'),
        IS_FINANCE_LCR = ISNULL(IS_FINANCE_LCR,'all FINANCE_LCR'),
        CLI_SUBTYPE = ISNULL(CLI_SUBTYPE,'ЮЛ');

    /*--------- 8. Загрузка в витрину ------------------------------*/
    IF OBJECT_ID('ALM_TEST.WORK.test_UL_cube_full') IS NULL
    BEGIN
        SELECT *, CAST(NULL AS datetime) AS DT_INSERT
        INTO   ALM_TEST.WORK.test_UL_cube_full
        FROM   #cube WHERE 1=0;
    END

    DELETE FROM ALM_TEST.WORK.test_UL_cube_full
    WHERE [Date] BETWEEN @dt_Start AND @dt_End;

    INSERT INTO ALM_TEST.WORK.test_UL_cube_full
    SELECT *, GETDATE() FROM #cube;
END
GO
```

### Что изменилось / добавилось

| Требование                     | Реализация                                                                                                                          |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| **ЮЛ Крупн./ССВ**              | `CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')`                                                                                             |
| **SEG\_NAME**                  | вытягивается, участвует в CUBE, `NULL→'all segment'`                                                                                |
| **TYPE ‘Начало’ + ‘Срез’**     | две ветки выборки: `DT_OPEN` (i=2) и `cal.Date` в интервале (i=1)                                                                   |
| **TERM\_GROUP (бакеты)**       | через `man_TermGroup`; идёт в CUBE                                                                                                  |
| **IS\_FINANCE\_LCR / IS\_PDR** | берутся из `dep`, участвуют в CUBE, `NULL→'all …'`; расчёт `UL_OUTFLOW_LCR` использует их                                           |
| **IS\_OPTION, MARGIN\_TYPE**   | оба в CUBE; `MARGIN_TYPE` рассчитывается на лету (PLUS / MINUS)                                                                     |
| **Баланс не дублируется**      | нет `UNION ALL` с самим собой; «all…» строки получаются только из `GROUP BY CUBE` — суммарный остаток на дату совпадает с деталями. |

Запусти:

```sql
EXEC work.prc_test_UL_cube_full
     @dt_Start = '2024-12-01',
     @dt_End   = '2025-06-26';
```

и проверь, что строки

* `[TYPE] = 'Срез' , CLI_SUBTYPE = 'ЮЛ', …` дают **точно тот же остаток**, что портфель-срез у коллег,
* `[TYPE] = 'Начало' , …` — те же объёмы «день-ко-дню»,
  а все нужные разрезы («all …») присутствуют.
