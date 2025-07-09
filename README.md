/*--------------------------------------------------------------------
  ДЕЛЬТА-СВЕРКА ДВУХ СНЕПШОТОВ DepositInterestsRateSnap
  ────────────────────────────────────────────────────────────────────
  • Берём два DT_REP:
        @rep_old = '2025-07-08'
        @rep_new = самый поздний DT_REP  >  @rep_old
  • В выборку попадают только те договоры, которые были «живыми»
        хотя бы один день в ИЮНЕ-2025:
           DT_OPEN ≤ 30-июн-2025  и  (DT_CLOSE > 01-июн-2025  или DT_CLOSE IS NULL)
  • Для каждого договора рассчитываем, по каким причинам он
        ИСКЛЮЧАЕТСЯ из витрины (если причин нет → 'OK').
  • Итоговый результат — список CON_ID, которые:
        – отсутствуют в более позднем снимке,
        – ИЛИ имеют другой набор причин-исключений.
--------------------------------------------------------------------*/
DECLARE @rep_old date = '2025-07-08',
        @rep_new date;

/* самый свежий DT_REP после @rep_old */
SELECT @rep_new = MAX(DT_REP)
FROM   [ALM_TEST].[WORK].[DepositInterestsRateSnap] WITH (NOLOCK)
WHERE  DT_REP > @rep_old;

IF @rep_new IS NULL
BEGIN
    RAISERROR (N'Более позднего DT_REP, чем %s, не найдено.', 11, 1, CONVERT(char(10),@rep_old,120));
    RETURN;
END
/*------------------------------------------------------------------*/
;WITH active_june AS (   /* два снимка, активные ≥ 1 дня в июне-25 */
    SELECT  d.CON_ID,
            d.DT_REP,
            d.CUR,
            d.CLI_SUBTYPE,
            ISNULL(d.isfloat,0)                    AS isfloat,
            d.MonthlyCONV_ALM_TransfertRate        AS tr_rate,
            d.MonthlyCONV_LIQ_TransfertRate,
            d.MonthlyCONV_RATE,
            d.LIQ_ФОР                              AS for_fact,
            d.IsDomRF,
            d.RATE,
            d.CLI_ID,
            d.MonthlyCONV_ForecastKeyRate          AS kbr_fore,
            d.ALM_OptionRate,
            d.IS_OPTION,
            d.DT_OPEN,
            d.DT_CLOSE
    FROM   [ALM_TEST].[WORK].[DepositInterestsRateSnap] d WITH (NOLOCK)
    WHERE  d.DT_REP IN (@rep_old,@rep_new)
      AND  d.DT_OPEN                       <= '2025-06-30'
      AND  ISNULL(d.DT_CLOSE,'9999-12-31')  >  '2025-06-01'
),
/*------------------------------------------------------------------
  вычисляем бинарные ФЛАГИ-ИСКЛЮЧЕНИЯ
------------------------------------------------------------------*/
flags AS (
    SELECT  a.*,
        /* 1 – исключаем, 0 – условие выполнено (=пройти фильтр) */
        CASE WHEN a.CUR = 'RUR'                              THEN 0 ELSE 1 END  AS f_CUR,
        CASE WHEN a.CLI_SUBTYPE = N'ФЛ'                      THEN 0 ELSE 1 END  AS f_CLI,
        CASE WHEN a.isfloat = 0                              THEN 0 ELSE 1 END  AS f_FLOAT,
        CASE WHEN a.tr_rate IS NOT NULL                      THEN 0 ELSE 1 END  AS f_NO_TRANS,
        CASE WHEN a.MonthlyCONV_RATE IS NOT NULL             THEN 0 ELSE 1 END  AS f_NO_RATE,
        CASE WHEN a.for_fact IS NOT NULL                     THEN 0 ELSE 1 END  AS f_NO_FOR,
        CASE WHEN a.IsDomRF = 0                              THEN 0 ELSE 1 END  AS f_DOMRF,
        CASE WHEN a.RATE > 0.01                              THEN 0 ELSE 1 END  AS f_RATE_LOW,
        CASE WHEN a.CLI_ID = 3731800                         THEN 1 ELSE 0 END  AS f_CLI3731800,
        CASE WHEN a.MonthlyCONV_RATE + a.ALM_OptionRate*a.IS_OPTION 
                 BETWEEN a.kbr_fore - 0.07 AND a.kbr_fore + 0.07
                                                           THEN 0 ELSE 1 END  AS f_SPREAD_OUT,
        CASE WHEN a.DT_OPEN = DATEADD(DAY,-1,a.DT_CLOSE)     THEN 1 ELSE 0 END  AS f_OPEN_EQ_CLOSE
    FROM active_june a
),
/*------------------------------------------------------------------
  превращаем набор флагов → строку причин («OK» если всё прошло)
------------------------------------------------------------------*/
reason_codes AS (
    SELECT  f.CON_ID,
            f.DT_REP,
            COALESCE(
              NULLIF(
                STUFF((
                    SELECT ','+code
                    FROM (VALUES
                        ('CUR'           ,f.f_CUR),
                        ('CLI'           ,f.f_CLI),
                        ('FLOAT'         ,f.f_FLOAT),
                        ('NO_TRANS'      ,f.f_NO_TRANS),
                        ('NO_RATE'       ,f.f_NO_RATE),
                        ('NO_FOR'        ,f.f_NO_FOR),
                        ('DOMRF'         ,f.f_DOMRF),
                        ('RATE_LOW'      ,f.f_RATE_LOW),
                        ('CLI_3731800'   ,f.f_CLI3731800),
                        ('SPREAD_OUT'    ,f.f_SPREAD_OUT),
                        ('OPEN_EQ_CLOSE' ,f.f_OPEN_EQ_CLOSE)
                    ) AS v(code,flag)
                    WHERE v.flag = 1
                    FOR XML PATH(''), TYPE
                ).value('.','nvarchar(max)'),1,1,''),''),'OK')   AS reason_set
    FROM flags f
),
old_snap AS ( SELECT CON_ID, reason_set FROM reason_codes WHERE DT_REP = @rep_old ),
new_snap AS ( SELECT CON_ID, reason_set FROM reason_codes WHERE DT_REP = @rep_new )
/*------------------------------------------------------------------
  ИТОГ: договора MISSING или с изменившимися флагами
------------------------------------------------------------------*/
SELECT  COALESCE(o.CON_ID, n.CON_ID)                  AS CON_ID,
        o.reason_set                                  AS reasons_old,
        n.reason_set                                  AS reasons_new,
        CASE 
            WHEN n.CON_ID IS NULL    THEN 'MISSING_IN_NEW'
            WHEN o.CON_ID IS NULL    THEN 'NEW_IN_NEW'
            WHEN o.reason_set <> n.reason_set
                                   THEN 'FLAGS_CHANGED'
        END                                           AS diff_status
FROM    old_snap o
FULL    JOIN new_snap n  ON n.CON_ID = o.CON_ID
WHERE   n.CON_ID IS NULL
     OR o.CON_ID IS NULL
     OR o.reason_set <> n.reason_set
ORDER BY diff_status, CON_ID
OPTION (RECOMPILE);
