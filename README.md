/*=====================================================================
  СВЕРКА “ОТПАВШИХ” ДЕПОЗИТОВ
  ---------------------------------------------------------------
  • Источник         : [LIQUIDITY].[liq].[DepositInterestsRate]   (НЕ snapshot)
  • Период снимков   : 05-июл-2025  vs.  последний DT_REP в таблице
  • Проверяем только договора, ЖИВЫЕ на 20-июн-2025
        DT_OPEN ≤ '2025-06-20'  and  DT_CLOSE > '2025-06-20'
  • Выводим договоры, у которых статус прохождения фильтров
        изменился между двумя репорт-датами,
        + причины отказа во fresh-снимке
=====================================================================*/
SET NOCOUNT ON;

DECLARE @analysis_date date = '2025-06-20';   /* дата жизнеспособности */
DECLARE @prev_rep      date = '2025-07-05';   /* «старый» DT_REP       */
DECLARE @latest_rep    date;                  /* самый свежий DT_REP   */

SELECT @latest_rep = MAX(DT_REP)
FROM   [LIQUIDITY].[liq].[DepositInterestsRate] WITH (NOLOCK);

IF @latest_rep = @prev_rep
BEGIN
    RAISERROR (N'Нет более свежего снимка, чем %s', 16, 1, @prev_rep);
    RETURN;
END;
;WITH
/*--------------------------------------------------------------------
  1.  Пул договоров для двух DT_REP  + остаток на @analysis_date
--------------------------------------------------------------------*/
base AS (
    SELECT  d.CON_ID,
            d.DT_REP,
            d.MonthlyCONV_ALM_TransfertRate,
            d.MonthlyCONV_LIQ_TransfertRate,
            d.LIQ_ФОР,
            d.MonthlyCONV_RATE,
            d.IsDomRF,
            d.RATE,
            d.ALM_OptionRate,
            d.IS_OPTION,
            d.MonthlyCONV_ForecastKeyRate,
            d.CLI_ID,
            d.DT_OPEN,
            d.DT_CLOSE,
            ISNULL(s.OUT_RUB,0) AS OUT_RUB
    FROM   [LIQUIDITY].[liq].[DepositInterestsRate] d  WITH (NOLOCK)
    LEFT   JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] s WITH (NOLOCK)
           ON  s.CON_ID = d.CON_ID
          AND @analysis_date BETWEEN s.DT_FROM AND s.DT_TO      -- остаток на 20-июн-25
    WHERE  d.DT_REP IN (@prev_rep, @latest_rep)
      AND  d.CLI_SUBTYPE      = N'ФЛ'
      AND  ISNULL(d.isfloat,0)= 0
      AND  d.CUR              = N'RUR'
      AND  d.DT_OPEN          <= @analysis_date                 -- жив на 20-июн-25
      AND  d.DT_CLOSE         >  @analysis_date
),
/*--------------------------------------------------------------------
  2.  Флаги нарушений (один столбец - один фильтр из процедуры)
--------------------------------------------------------------------*/
tag AS (
    SELECT  b.*,
            /*1*/ CASE WHEN b.MonthlyCONV_ALM_TransfertRate IS NULL                    THEN 1 ELSE 0 END AS f_transfert_null,
            /*2*/ CASE WHEN b.LIQ_ФОР IS NULL                                          THEN 1 ELSE 0 END AS f_for_null,
            /*3*/ CASE WHEN b.MonthlyCONV_RATE IS NULL                                 THEN 1 ELSE 0 END AS f_rate_null,
            /*4*/ CASE WHEN b.IsDomRF <> 0                                             THEN 1 ELSE 0 END AS f_domrf,
            /*5*/ CASE WHEN b.RATE <= 0.01                                             THEN 1 ELSE 0 END AS f_rate_small,
            /*6*/ CASE WHEN b.MonthlyCONV_ForecastKeyRate IS NULL
                       OR b.MonthlyCONV_RATE + ISNULL(b.ALM_OptionRate,0)*ISNULL(b.IS_OPTION,0)
                          NOT BETWEEN b.MonthlyCONV_ForecastKeyRate - 0.07
                              AND   b.MonthlyCONV_ForecastKeyRate + 0.07               THEN 1 ELSE 0 END AS f_spread,
            /*7*/ CASE WHEN b.CLI_ID = 3731800                                         THEN 1 ELSE 0 END AS f_cli_ban,
            /*8*/ CASE WHEN b.DT_OPEN = DATEADD(DAY,-1,b.DT_CLOSE)                     THEN 1 ELSE 0 END AS f_one_day,
            /*9*/ CASE WHEN ISNULL(b.OUT_RUB,0) = 0                                    THEN 1 ELSE 0 END AS f_saldo_zero
    FROM   base b
),
/*--------------------------------------------------------------------
  3.  Итоговый статус «прошёл все фильтры»
--------------------------------------------------------------------*/
pass AS (
    SELECT  t.*,
            CASE WHEN t.f_transfert_null
                    | t.f_for_null
                    | t.f_rate_null
                    | t.f_domrf
                    | t.f_rate_small
                    | t.f_spread
                    | t.f_cli_ban
                    | t.f_one_day
                    | t.f_saldo_zero = 0 THEN 1 ELSE 0 END AS pass_all
    FROM tag t
),
/*--------------------------------------------------------------------
  4.  Статусы на старый и свежий репорты
--------------------------------------------------------------------*/
prev   AS (SELECT CON_ID, pass_all FROM pass WHERE DT_REP = @prev_rep  ),
latest AS (SELECT CON_ID, pass_all FROM pass WHERE DT_REP = @latest_rep),
diff   AS (
    SELECT COALESCE(p.CON_ID,l.CON_ID) AS CON_ID,
           ISNULL(p.pass_all,0)        AS pass_prev,
           ISNULL(l.pass_all,0)        AS pass_latest
    FROM prev   p
    FULL JOIN latest l ON l.CON_ID = p.CON_ID
),
/*--------------------------------------------------------------------
  5.  Детализация причин отказа во fresh-снимке
--------------------------------------------------------------------*/
reasons AS (
    SELECT CON_ID, 'TRANSFERT_NULL'  AS rs FROM tag WHERE DT_REP=@latest_rep AND f_transfert_null=1
    UNION ALL
    SELECT CON_ID, 'FOR_NULL'        FROM tag WHERE DT_REP=@latest_rep AND f_for_null       =1
    UNION ALL
    SELECT CON_ID, 'RATE_NULL'       FROM tag WHERE DT_REP=@latest_rep AND f_rate_null      =1
    UNION ALL
    SELECT CON_ID, 'IsDomRF<>0'      FROM tag WHERE DT_REP=@latest_rep AND f_domrf          =1
    UNION ALL
    SELECT CON_ID, 'RATE<=0.01'      FROM tag WHERE DT_REP=@latest_rep AND f_rate_small     =1
    UNION ALL
    SELECT CON_ID, 'SPREAD_OUTSIDE'  FROM tag WHERE DT_REP=@latest_rep AND f_spread         =1
    UNION ALL
    SELECT CON_ID, 'CLI_ID_3731800'  FROM tag WHERE DT_REP=@latest_rep AND f_cli_ban        =1
    UNION ALL
    SELECT CON_ID, 'ONE_DAY_DEAL'    FROM tag WHERE DT_REP=@latest_rep AND f_one_day        =1
    UNION ALL
    SELECT CON_ID, 'SALDO_ZERO'      FROM tag WHERE DT_REP=@latest_rep AND f_saldo_zero     =1
)
/*--------------------------------------------------------------------
  6.  ФИНАЛ: договоры, у которых pass_prev ≠ pass_latest
--------------------------------------------------------------------*/
SELECT  d.CON_ID,
        d.pass_prev,
        d.pass_latest,
        STRING_AGG(r.rs, '; ') WITHIN GROUP (ORDER BY r.rs) AS fail_reasons_latest
FROM    diff d
LEFT JOIN reasons r ON r.CON_ID = d.CON_ID
WHERE   d.pass_prev <> d.pass_latest         -- только изменившиеся
GROUP BY d.CON_ID, d.pass_prev, d.pass_latest
ORDER BY d.pass_prev DESC, d.pass_latest, d.CON_ID
OPTION (RECOMPILE);
