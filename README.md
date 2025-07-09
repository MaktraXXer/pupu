/*---------------------------------------------------------------
  СВЕРКА ПОРТФЕЛЯ «LIQUIDITY» vs. фильтры процедуры
  Период          : 18-июн-2025 … 07-июл-2025
  Срез портфеля   : последний DT_REP ( = @latest_rep )
  Показатели      : - кол-во договоров, - сумма OUT_RUB
  ---------------------------------------------------------------*/
USE [ALM_TEST];
SET NOCOUNT ON;
GO
------------------------------------------------------------------
DECLARE @dt_from     date = '2025-06-18',
        @dt_to       date = '2025-07-07',
        @latest_rep  date;

SELECT @latest_rep = MAX(DT_REP)
FROM   [LIQUIDITY].[liq].[DepositInterestsRate] WITH (NOLOCK);
------------------------------------------------------------------
;WITH
/* календарь запрашиваемого периода */
cal AS (
    SELECT @dt_from AS calc_date
    UNION ALL
    SELECT DATEADD(DAY,1,calc_date)
    FROM   cal
    WHERE  calc_date < @dt_to
),
/*--------------------------------------------------------------
  Портфель из LIQUIDITY (базовые фильтры)
----------------------------------------------------------------*/
liq_dep AS (
    SELECT d.CON_ID,
           d.BALANCE_RUB,
           d.DT_OPEN,
           d.DT_CLOSE
    FROM   [LIQUIDITY].[liq].[DepositInterestsRate] d WITH (NOLOCK)
    WHERE  d.DT_REP      = @latest_rep
      AND  d.CLI_SUBTYPE = N'ФЛ'
      AND  ISNULL(d.isfloat,0) = 0
      AND  d.CUR         = N'RUR'
),
liq_active AS (
    SELECT  c.calc_date,
            l.CON_ID
    FROM    liq_dep  l
    JOIN    cal      c ON c.calc_date >= l.DT_OPEN
                      AND c.calc_date <  ISNULL(l.DT_CLOSE,'9999-12-31')
),
saldo_liq AS (
    SELECT  s.CON_ID,
            c.calc_date,
            SUM(s.OUT_RUB) AS OUT_RUB          -- одна строка на договор/дату
    FROM   [LIQUIDITY].[liq].[DepositContract_Saldo] s WITH (NOLOCK)
    JOIN   cal c ON c.calc_date BETWEEN s.DT_FROM AND s.DT_TO
    GROUP BY s.CON_ID, c.calc_date
),
liq_agg AS (
    SELECT la.calc_date,
           COUNT(DISTINCT la.CON_ID)                          AS cnt_liq,
           SUM(ISNULL(sl.OUT_RUB,0))                          AS bal_liq
    FROM   liq_active   la
    LEFT  JOIN saldo_liq sl
           ON sl.CON_ID    = la.CON_ID
          AND sl.calc_date = la.calc_date
    GROUP BY la.calc_date
),
/*--------------------------------------------------------------
  Портфель из DepositInterestsRateSnap (фильтры процедуры)
----------------------------------------------------------------*/
snap_dep AS (
    SELECT dep.CON_ID,
           dep.BALANCE_RUB,
           dep.DT_OPEN,
           dep.DT_CLOSE
    FROM   [ALM_TEST].[WORK].[DepositInterestsRateSnap] dep WITH (NOLOCK)
    WHERE  dep.DT_REP = @latest_rep
      AND  dep.[CLI_SUBTYPE]   = N'ФЛ'
      AND  ISNULL(dep.[isfloat],0)        = 0
      AND  dep.[CUR]                      = N'RUR'
      AND  dep.[MonthlyCONV_ALM_TransfertRate] IS NOT NULL
      AND  ISNULL(dep.[MonthlyCONV_ALM_TransfertRate],dep.[MonthlyCONV_LIQ_TransfertRate]) IS NOT NULL
      AND  dep.[LIQ_ФОР] IS NOT NULL
      AND  dep.[MonthlyCONV_RATE] IS NOT NULL
      AND  dep.[IsDomRF]       = 0
      AND  dep.[RATE]          > 0.01
      AND  dep.[MonthlyCONV_RATE] + dep.[ALM_OptionRate]*dep.[IS_OPTION]
           BETWEEN dep.[MonthlyCONV_ForecastKeyRate] - 0.07
               AND dep.[MonthlyCONV_ForecastKeyRate] + 0.07
      AND  dep.[CLI_ID]        <> 3731800
      AND  dep.[DT_OPEN]       <> DATEADD(DAY,-1,dep.[DT_CLOSE])
),
snap_active AS (
    SELECT  c.calc_date,
            s.CON_ID
    FROM    snap_dep s
    JOIN    cal      c ON c.calc_date >= s.DT_OPEN
                      AND c.calc_date <  ISNULL(s.DT_CLOSE,'9999-12-31')
),
saldo_snap AS (
    SELECT  sd.CON_ID,
            c.calc_date,
            SUM(sd.OUT_RUB) AS OUT_RUB
    FROM   [LIQUIDITY].[liq].[DepositContract_Saldo] sd WITH (NOLOCK)
    JOIN   cal c ON c.calc_date BETWEEN sd.DT_FROM AND sd.DT_TO
    GROUP BY sd.CON_ID, c.calc_date
),
snap_agg AS (
    SELECT sa.calc_date,
           COUNT(DISTINCT sa.CON_ID)                          AS cnt_snap,
           SUM(ISNULL(ss.OUT_RUB,0))                          AS bal_snap
    FROM   snap_active sa
    LEFT  JOIN saldo_snap ss
           ON ss.CON_ID    = sa.CON_ID
          AND ss.calc_date = sa.calc_date
    GROUP BY sa.calc_date
)
/*--------------------------------------------------------------
  Итоговая сверка
----------------------------------------------------------------*/
SELECT  c.calc_date                                              AS report_date,
        ISNULL(l.cnt_liq,0)      AS cnt_liq,
        ISNULL(l.bal_liq,0)      AS out_rub_liq,
        ISNULL(s.cnt_snap,0)     AS cnt_proc,
        ISNULL(s.bal_snap,0)     AS out_rub_proc,
        ISNULL(l.cnt_liq,0) - ISNULL(s.cnt_snap,0)    AS cnt_diff,
        ISNULL(l.bal_liq,0) - ISNULL(s.bal_snap,0)    AS out_rub_diff
FROM    cal       c
LEFT JOIN liq_agg l ON l.calc_date = c.calc_date
LEFT JOIN snap_agg s ON s.calc_date = c.calc_date
ORDER BY c.calc_date
OPTION (MAXRECURSION 1000, RECOMPILE);

/*--------------------------------------------------------------
---- ДЕТАЛИ по несовпадающим договорам (необязательно):
----  раскомментируйте, если нужна полная выборка дисперсии
------------------------------------------------------------------
-- договора, которые есть в LIQUIDITY, но отсутствуют после фильтров
-- SELECT la.calc_date, la.CON_ID
-- FROM   liq_active la
-- EXCEPT
-- SELECT sa.calc_date, sa.CON_ID
-- FROM   snap_active sa;

-- договора, которые прошли фильтры, но отсутствуют в базовом портфеле
-- SELECT sa.calc_date, sa.CON_ID
-- FROM   snap_active sa
-- EXCEPT
-- SELECT la.calc_date, la.CON_ID
-- FROM   liq_active la;
------------------------------------------------------------------*/
