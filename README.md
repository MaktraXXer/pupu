```sql
/*===================================================================
   КАКИЕ ДОГОВОРЫ “ОТВАЛИЛИСЬ” С 08-ИЮЛ-2025 ⇒ LATEST
   - и по какому именно фильтру
   ---------------------------------------------------------------
   • База-источник :  ALM_TEST.WORK.DepositInterestsRateSnap
   • Фильтры       :  полный набор из prc_GenerateGroupDepositInterestsRateVer1
   • Итог-1        :  список CON_ID + перечисление причин исключения
   • Итог-2        :  сводка количества договоров по каждой причине
   ----------------------------------------------------------------*/
DECLARE @prev_rep   date = '2025-07-08';                    -- «старый» снимок
DECLARE @latest_rep date;                                   -- самый свежий

SELECT @latest_rep = MAX(DT_REP)
FROM   [ALM_TEST].[WORK].[DepositInterestsRateSnap] WITH (NOLOCK);

IF @latest_rep = @prev_rep
BEGIN
    RAISERROR('Нет более свежего снимка, чем %s', 16, 1, @prev_rep);
    RETURN;
END;
;WITH
/*---------------------------------------------------------------
  1.  Базовый пул с мягкими фильтрами (общими для обоих снимков)
----------------------------------------------------------------*/
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
            d.DT_CLOSE
    FROM  [ALM_TEST].[WORK].[DepositInterestsRateSnap] d WITH (NOLOCK)
    WHERE d.DT_REP IN (@prev_rep, @latest_rep)
      AND d.CLI_SUBTYPE      = N'ФЛ'
      AND ISNULL(d.isfloat,0)= 0
      AND d.CUR              = N'RUR'
),
/*----------------------------------------------------------------
  2.  Размечаем каждый «жёсткий» фильтр отдельным флагом
----------------------------------------------------------------*/
tag AS (
    SELECT  b.*,
            /* 1 */ CASE WHEN b.MonthlyCONV_ALM_TransfertRate IS NULL                   THEN 1 ELSE 0 END AS flg_transfert_null,
            /* 2 */ CASE WHEN b.LIQ_ФОР IS NULL                                         THEN 1 ELSE 0 END AS flg_for_null,
            /* 3 */ CASE WHEN b.MonthlyCONV_RATE IS NULL                                THEN 1 ELSE 0 END AS flg_rate_null,
            /* 4 */ CASE WHEN b.IsDomRF <> 0                                            THEN 1 ELSE 0 END AS flg_domrf,
            /* 5 */ CASE WHEN b.RATE <= 0.01                                            THEN 1 ELSE 0 END AS flg_rate_small,
            /* 6 */ CASE WHEN ( b.MonthlyCONV_ForecastKeyRate IS NULL
                               OR b.MonthlyCONV_RATE 
                                  + ISNULL(b.ALM_OptionRate,0)*ISNULL(b.IS_OPTION,0)
                                     NOT BETWEEN b.MonthlyCONV_ForecastKeyRate - 0.07
                                         AND     b.MonthlyCONV_ForecastKeyRate + 0.07)  THEN 1 ELSE 0 END AS flg_spread,
            /* 7 */ CASE WHEN b.CLI_ID = 3731800                                        THEN 1 ELSE 0 END AS flg_cli_ban,
            /* 8 */ CASE WHEN b.DT_OPEN = DATEADD(DAY,-1,b.DT_CLOSE)                    THEN 1 ELSE 0 END AS flg_one_day
    FROM base b
),
/*----------------------------------------------------------------
  3.  Итоговый признак «прошёл все фильтры»
----------------------------------------------------------------*/
pass AS (
    SELECT  t.*,
            CASE WHEN t.flg_transfert_null
                    | t.flg_for_null
                    | t.flg_rate_null
                    | t.flg_domrf
                    | t.flg_rate_small
                    | t.flg_spread
                    | t.flg_cli_ban
                    | t.flg_one_day = 0 THEN 1 ELSE 0 END AS pass_all
    FROM tag t
),
prev_ok   AS (SELECT CON_ID FROM pass WHERE DT_REP = @prev_rep   AND pass_all = 1),
latest_ok AS (SELECT CON_ID FROM pass WHERE DT_REP = @latest_rep AND pass_all = 1),
lost_ids  AS (SELECT CON_ID FROM prev_ok EXCEPT SELECT CON_ID FROM latest_ok),
/*----------------------------------------------------------------
  4.  Подробности по «потерянным» договорам
----------------------------------------------------------------*/
lost_det AS (
    SELECT  l.CON_ID,
            CASE WHEN t.DT_REP IS NULL THEN 1 ELSE 0 END                     AS absent_latest,
            ISNULL(t.flg_transfert_null,0)                                   AS flg_transfert_null,
            ISNULL(t.flg_for_null,0)                                         AS flg_for_null,
            ISNULL(t.flg_rate_null,0)                                        AS flg_rate_null,
            ISNULL(t.flg_domrf,0)                                            AS flg_domrf,
            ISNULL(t.flg_rate_small,0)                                       AS flg_rate_small,
            ISNULL(t.flg_spread,0)                                           AS flg_spread,
            ISNULL(t.flg_cli_ban,0)                                          AS flg_cli_ban,
            ISNULL(t.flg_one_day,0)                                          AS flg_one_day
    FROM    lost_ids l
    LEFT JOIN tag t
           ON t.CON_ID = l.CON_ID
          AND t.DT_REP = @latest_rep
),
/*----------------------------------------------------------------
  5-A.  Расшифровка: CON_ID  ⇒  причины исключения
----------------------------------------------------------------*/
reasons AS (
    SELECT  ld.CON_ID,
            CASE WHEN ld.absent_latest      = 1 THEN 'ABSENT_LATEST'      END AS reason
    FROM    lost_det ld
    UNION ALL SELECT CON_ID, 'TRANSFERT_NULL'  FROM lost_det WHERE flg_transfert_null = 1
    UNION ALL SELECT CON_ID, 'FOR_NULL'        FROM lost_det WHERE flg_for_null       = 1
    UNION ALL SELECT CON_ID, 'RATE_NULL'       FROM lost_det WHERE flg_rate_null      = 1
    UNION ALL SELECT CON_ID, 'IsDomRF<>0'      FROM lost_det WHERE flg_domrf          = 1
    UNION ALL SELECT CON_ID, 'RATE<=0.01'      FROM lost_det WHERE flg_rate_small     = 1
    UNION ALL SELECT CON_ID, 'SPREAD_OUTSIDE'  FROM lost_det WHERE flg_spread         = 1
    UNION ALL SELECT CON_ID, 'CLI_ID_3731800'  FROM lost_det WHERE flg_cli_ban        = 1
    UNION ALL SELECT CON_ID, 'ONE_DAY_DEAL'    FROM lost_det WHERE flg_one_day        = 1
)
/*----------------------------------------------------------------
  5-B.  Итог-1 : детальный список договоров + причины
----------------------------------------------------------------*/
SELECT  r.CON_ID,
        STRING_AGG(r.reason, '; ') WITHIN GROUP (ORDER BY r.reason) AS fail_reasons
FROM    reasons r
GROUP BY r.CON_ID
ORDER BY r.CON_ID
OPTION (RECOMPILE);

/*----------------------------------------------------------------
  6.  Итог-2 : сводка количества CON_ID по каждой причине
----------------------------------------------------------------*/
SELECT  r.reason,
        COUNT(DISTINCT r.CON_ID) AS lost_count
FROM    reasons r
GROUP BY r.reason
ORDER BY r.reason
OPTION (RECOMPILE);
```
