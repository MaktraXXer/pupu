```sql
/*-----------------------------------------------------------------------------
  Список срочных депозитов-ФЛ, живших в июне-2025,
  прошедших ВСЕ боевые фильтры в «старом» снимке 05-июл-2025,
  и их статус в самом свежем снимке (есть ли строка и заполнен ли
  MonthlyCONV_ALM_TransfertRate).
  ---------------------------------------------------------------------------
  • Источник данных : [LIQUIDITY].[liq].[DepositInterestsRate]  (NOLOCK)
  • Фильтры        :
        CLI_SUBTYPE            = 'ФЛ'
        isfloat                = 0 / NULL
        CUR                    = 'RUR'
        MATUR > 0              — срочный вклад
        LIQ_ФОР, MonthlyCONV_RATE            NOT NULL
        IsDomRF               = 0
        RATE                  > 0.01
        Forecast-спред        ±0.07 p.p.
        CLI_ID               <> 3731800
        DT_OPEN <> DT_CLOSE-1
  • «Живой» в июне-2025        : DT_OPEN ≤ '2025-06-30'  and  DT_CLOSE > '2025-06-01'
  • «Старый» снимок            : 05-июл-2025 → @old_rep
  • «Новый» снимок             : MAX(DT_REP)  → @new_rep
-----------------------------------------------------------------------------*/
SET NOCOUNT ON;
DECLARE @old_rep  date = '2025-07-05';   -- старый DT_REP
DECLARE @new_rep  date  = (SELECT MAX(DT_REP)
                           FROM  [LIQUIDITY].[liq].[DepositInterestsRate] WITH (NOLOCK));

/* контроль: на всякий случай */
IF @new_rep = @old_rep
BEGIN
    RAISERROR (N'Нет более свежего DT_REP, чем %s', 16, 1, @old_rep);
    RETURN;
END;

/* границы «живости» в июне-2025 */
DECLARE @june_start date = '2025-06-01',
        @june_end   date = '2025-06-30';

;WITH
/*---------------------------------------------------------------------------
  1.  Договоры из старого снимка, прошедшие ВСЕ фильтры
---------------------------------------------------------------------------*/
old_dep AS (
    SELECT  d.CON_ID,
            d.MonthlyCONV_ALM_TransfertRate AS old_transfer
    FROM   [LIQUIDITY].[liq].[DepositInterestsRate] d WITH (NOLOCK)
    WHERE  d.DT_REP               =  @old_rep
      AND  d.CLI_SUBTYPE          =  N'ФЛ'
      AND  ISNULL(d.isfloat,0)    =  0
      AND  d.CUR                  =  N'RUR'
      AND  d.MATUR               >  0
      AND  d.LIQ_ФОР              IS NOT NULL
      AND  d.MonthlyCONV_RATE     IS NOT NULL
      AND  d.MonthlyCONV_ALM_TransfertRate IS NOT NULL
      AND  d.IsDomRF              =  0
      AND  d.RATE                >  0.01
      AND  d.MonthlyCONV_ForecastKeyRate IS NOT NULL
      AND  d.MonthlyCONV_RATE
           + ISNULL(d.ALM_OptionRate,0)*ISNULL(d.IS_OPTION,0)
           BETWEEN d.MonthlyCONV_ForecastKeyRate - 0.07
               AND d.MonthlyCONV_ForecastKeyRate + 0.07
      AND  d.CLI_ID              <>  3731800
      AND  d.DT_OPEN             <>  DATEADD(DAY,-1,d.DT_CLOSE)
      AND  d.DT_OPEN             <=  @june_end               -- жил в июне-25
      AND  d.DT_CLOSE            >   @june_start
),
/*---------------------------------------------------------------------------
  2.  Те же CON_ID в самом свежем снимке (фильтры те же, кроме
      допускаем NULL в MonthlyCONV_ALM_TransfertRate)
---------------------------------------------------------------------------*/
new_dep AS (
    SELECT  d.CON_ID,
            d.MonthlyCONV_ALM_TransfertRate AS new_transfer
    FROM   [LIQUIDITY].[liq].[DepositInterestsRate] d WITH (NOLOCK)
    WHERE  d.DT_REP               =  @new_rep
      AND  d.CLI_SUBTYPE          =  N'ФЛ'
      AND  ISNULL(d.isfloat,0)    =  0
      AND  d.CUR                  =  N'RUR'
      AND  d.MATUR               >  0
      AND  d.LIQ_ФОР              IS NOT NULL
      AND  d.MonthlyCONV_RATE     IS NOT NULL
      /* —--- нет проверки на NOT NULL для new_transfer --- */
      AND  d.IsDomRF              =  0
      AND  d.RATE                >  0.01
      AND  d.MonthlyCONV_ForecastKeyRate IS NOT NULL
      AND  d.MonthlyCONV_RATE
           + ISNULL(d.ALM_OptionRate,0)*ISNULL(d.IS_OPTION,0)
           BETWEEN d.MonthlyCONV_ForecastKeyRate - 0.07
               AND d.MonthlyCONV_ForecastKeyRate + 0.07
      AND  d.CLI_ID              <>  3731800
      AND  d.DT_OPEN             <>  DATEADD(DAY,-1,d.DT_CLOSE)
      AND  d.DT_OPEN             <=  @june_end
      AND  d.DT_CLOSE            >   @june_start
)
/*---------------------------------------------------------------------------
  3.  Итог: статус наличия и значение поля в новом снимке
---------------------------------------------------------------------------*/
SELECT  o.CON_ID,
        CASE WHEN n.CON_ID IS NULL THEN 'нет в новом'
             ELSE 'есть' END                         AS exists_in_latest,
        n.new_transfer                               AS MonthlyCONV_ALM_TransfertRate_latest
FROM    old_dep  o
LEFT JOIN new_dep n
       ON n.CON_ID = o.CON_ID
ORDER BY exists_in_latest DESC, o.CON_ID;
```
