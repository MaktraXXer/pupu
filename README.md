/*=====================================================================
  КАКИЕ ДЕПОЗИТЫ «ОТВАЛИЛИСЬ» МЕЖДУ СНЭПШОТАМИ 05-ИЮЛ-2025 и LATEST
  - и КАКОЙ ФИЛЬТР ИХ ОТСЁК
  --------------------------------------------------------------------
  • Анализируем только договоры, живые на 20-июн-2025:
        DT_OPEN ≤ '2025-06-20'  and  DT_CLOSE > '2025-06-20'
  • Фильтры – ровно те же, что в  WORK.prc_GenerateGroupDepositInterestsRateVer1
  • Вывод:
        - pass_prev      – прошёл ли договор все фильтры 05-июл-2025
        - pass_latest    – прошёл ли договор все фильтры в свежем снимке
        - fail_reasons_latest – какие именно проверки «завалил» во fresh-репе
  -------------------------------------------------------------------*/
DECLARE @analysis_date date = '2025-06-20';   -- дата жизнеспособности
DECLARE @prev_rep      date = '2025-07-05';   -- старый снимок
DECLARE @latest_rep    date;                  -- самый свежий снимок

SELECT @latest_rep = MAX(DT_REP)
FROM   [ALM_TEST].[WORK].[DepositInterestsRateSnap] WITH (NOLOCK);

IF @latest_rep = @prev_rep
BEGIN
    RAISERROR('Нет более свежего снимка, чем %s', 16, 1, @prev_rep);
    RETURN;
END;
;WITH
/*--------------------------------------------------------------------
  1.  Пул договоров для двух снимков + остаток на 20-июн-2025
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
    FROM  [ALM_TEST].[WORK].[DepositInterestsRateSnap] d WITH (NOLOCK)
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] s WITH (NOLOCK)
           ON s.CON_ID = d.CON_ID
          AND @analysis_date BETWEEN s.DT_FROM AND s.DT_TO          -- остаток именно на 20-июн-2025
    WHERE d.DT_REP IN (@prev_rep, @latest_rep)
      AND d.CLI_SUBTYPE      = N'ФЛ'
      AND ISNULL(d.isfloat,0)= 0
      AND d.CUR              = N'RUR'
      AND d.DT_OPEN          <= @analysis_date                     -- жив на 20-июн
      AND d.DT_CLOSE         >  @analysis_date
),
/*--------------------------------------------------------------------
  2.  Отмечаем каждый «жёсткий» фильтр отдельным флагом
--------------------------------------------------------------------*/
tag AS (
    SELECT  b.*,
            /* 1 */ CASE WHEN b.MonthlyCONV_ALM_TransfertRate IS NULL                    THEN 1 ELSE 0 END AS flg_transfert_null,
            /* 2 */ CASE WHEN b.LIQ_ФОР IS NULL                                          THEN 1 ELSE 0 END AS flg_for_null,
            /* 3 */ CASE WHEN b.MonthlyCONV_RATE IS NULL                                 THEN 1 ELSE 0 END AS flg_rate_null,
            /* 4 */ CASE WHEN b.IsDomRF <> 0                                             THEN 1 ELSE 0 END AS flg_domrf,
            /* 5 */ CASE WHEN b.RATE <= 0.01                                             THEN 1 ELSE 0 END AS flg_rate_small,
            /* 6 */ CASE WHEN b.MonthlyCONV_ForecastKeyRate IS NULL
                           OR b.MonthlyCONV_RATE + ISNULL(b.ALM_OptionRate,0)*ISNULL(b.IS_OPTION,0)
                              NOT BETWEEN b.MonthlyCONV_ForecastKeyRate - 0.07
                                  AND   b.MonthlyCONV_ForecastKeyRate + 0.07             THEN 1 ELSE 0 END AS flg_spread,
            /* 7 */ CASE WHEN b.CLI_ID = 3731800                                         THEN 1 ELSE 0 END AS flg_cli_ban,
            /* 8 */ CASE WHEN b.DT_OPEN = DATEADD(DAY,-1,b.DT_CLOSE)                     THEN 1 ELSE 0 END AS flg_one_day,
            /* 9 */ CASE WHEN ISNULL(b.OUT_RUB,0) = 0                                    THEN 1 ELSE 0 END AS flg_saldo_zero
    FROM   base b
),
/*--------------------------------------------------------------------
  3.  Финальный признак «прошёл все фильтры»
--------------------------------------------------------------------*/
pass AS (
    SELECT  t.*,
            CASE WHEN t.flg_transfert_null
                    | t.flg_for_null
                    | t.flg_rate_null
                    | t.flg_domrf
                    | t.flg_rate_small
                    | t.flg_spread
                    | t.flg_cli_ban
                    | t.flg_one_day
                    | t.flg_saldo_zero = 0 THEN 1 ELSE 0 END AS pass_all
    FROM tag t
),
/*--------------------------------------------------------------------
  4.  Итоговые статусы на 05-июл и LATEST
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
  5.  Причины отказа во fresh-репе
--------------------------------------------------------------------*/
reasons AS (
    SELECT CON_ID,
           CASE WHEN flg_transfert_null = 1 THEN 'TRANSFERT_NULL'   END AS rs
    FROM tag WHERE DT_REP = @latest_rep
    UNION ALL SELECT CON_ID, 'FOR_NULL'       FROM tag WHERE DT_REP=@latest_rep AND flg_for_null       =1
    UNION ALL SELECT CON_ID, 'RATE_NULL'      FROM tag WHERE DT_REP=@latest_rep AND flg_rate_null      =1
    UNION ALL SELECT CON_ID, 'IsDomRF<>0'     FROM tag WHERE DT_REP=@latest_rep AND flg_domrf          =1
    UNION ALL SELECT CON_ID, 'RATE<=0.01'     FROM tag WHERE DT_REP=@latest_rep AND flg_rate_small     =1
    UNION ALL SELECT CON_ID, 'SPREAD_OUTSIDE' FROM tag WHERE DT_REP=@latest_rep AND flg_spread         =1
    UNION ALL SELECT CON_ID, 'CLI_ID_3731800' FROM tag WHERE DT_REP=@latest_rep AND flg_cli_ban        =1
    UNION ALL SELECT CON_ID, 'ONE_DAY_DEAL'   FROM tag WHERE DT_REP=@latest_rep AND flg_one_day        =1
    UNION ALL SELECT CON_ID, 'SALDO_ZERO'     FROM tag WHERE DT_REP=@latest_rep AND flg_saldo_zero     =1
)
/*--------------------------------------------------------------------
  6.  ФИНАЛ: только те договоры, у которых pass_prev ≠ pass_latest
--------------------------------------------------------------------*/
SELECT  d.CON_ID,
        d.pass_prev,
        d.pass_latest,
        STRING_AGG(r.rs,'; ') WITHIN GROUP (ORDER BY r.rs) AS fail_reasons_latest
FROM    diff d
LEFT JOIN reasons r ON r.CON_ID = d.CON_ID
WHERE   d.pass_prev <> d.pass_latest          -- показываем только «изменившихся»
GROUP BY d.CON_ID, d.pass_prev, d.pass_latest
ORDER BY d.pass_prev DESC, d.pass_latest, d.CON_ID
OPTION (RECOMPILE);
