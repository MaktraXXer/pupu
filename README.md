/*--------------------------------------------------------------------
  Ежедневная статистика по активным договорам
  Период: 18-июн-2025 … 07-июл-2025              (измените при-необходимости)
  Снимки депозита: 08-июл-2025  +  самый-последний DT_REP
  Фильтры: CLI_SUBTYPE='ФЛ', isfloat<>1, CUR='RUR'
  Договор считается «живым» на дату D, если DT_OPEN ≤ D  и (DT_CLOSE > D  или DT_CLOSE IS NULL)
  OUT_RUB подтягивается из DepositContract_Saldo; считаем также договора без saldo
--------------------------------------------------------------------*/
;WITH
latest_rep AS (               /* самый свежий DT_REP - фиксируем один раз */
    SELECT MAX(DT_REP) AS DT_REP
    FROM  [LIQUIDITY].[liq].[DepositInterestsRate] WITH (NOLOCK)
),
cal AS (                       /* календарь 18-июн-2025 … 07-июл-2025  */
    SELECT CAST('2025-06-18' AS date) AS calc_date
    UNION ALL
    SELECT DATEADD(DAY,1,calc_date)
    FROM   cal
    WHERE  calc_date < '2025-07-07'
),
/* два снимка депозитов с нужными фильтрами */
dep_filtered AS (
    /* --- 08-июл-2025 --- */
    SELECT  d.CON_ID,
            d.BALANCE_RUB,
            d.DT_OPEN,
            d.DT_CLOSE,
            'REP_20250708' AS rep_flag
    FROM  [LIQUIDITY].[liq].[DepositInterestsRate] d WITH (NOLOCK)
    WHERE d.DT_REP      = '2025-07-08'
      AND d.CLI_SUBTYPE = N'ФЛ'
      AND d.isfloat    <> 1
      AND d.CUR         = N'RUR'

    UNION ALL

    /* --- LATEST (если он НЕ 08-июл-2025) --- */
    SELECT  d.CON_ID,
            d.BALANCE_RUB,
            d.DT_OPEN,
            d.DT_CLOSE,
            'REP_LATEST'  AS rep_flag
    FROM  [LIQUIDITY].[liq].[DepositInterestsRate] d WITH (NOLOCK)
    CROSS JOIN latest_rep lr
    WHERE lr.DT_REP    <> '2025-07-08'     /* отсекаем дубли, если latest = 08-июл */
      AND d.DT_REP      = lr.DT_REP
      AND d.CLI_SUBTYPE = N'ФЛ'
      AND d.isfloat    <> 1
      AND d.CUR         = N'RUR'
),
/* договора, которые живы на каждую дату из календаря */
dep_active AS (
    SELECT  c.calc_date,
            df.rep_flag,
            df.CON_ID,
            df.BALANCE_RUB
    FROM    dep_filtered df
    JOIN    cal          c
           ON c.calc_date >= df.DT_OPEN
          AND c.calc_date <  ISNULL(df.DT_CLOSE,'9999-12-31')
),
/* saldo: одна строка на con_id/дату */
saldo_on_date AS (
    SELECT  s.CON_ID,
            c.calc_date,
            SUM(s.OUT_RUB) AS OUT_RUB
    FROM  [LIQUIDITY].[liq].[DepositContract_Saldo] s WITH (NOLOCK)
    JOIN  cal c   ON c.calc_date BETWEEN s.DT_FROM AND s.DT_TO
    GROUP BY s.CON_ID, c.calc_date
),
/* агрегаты по каждому снимку на каждую дату */
agg AS (
    SELECT  da.calc_date,
            da.rep_flag,
            COUNT(*)                                            AS cnt_dep,
            SUM(da.BALANCE_RUB)                                 AS sum_balance_rub,
            SUM(ISNULL(sd.OUT_RUB,0))                           AS sum_out_rub,
            SUM(CASE WHEN sd.CON_ID IS NULL THEN 1 ELSE 0 END)  AS cnt_no_saldo
    FROM      dep_active     da
    LEFT JOIN saldo_on_date  sd
           ON sd.CON_ID    = da.CON_ID
          AND sd.calc_date = da.calc_date
    GROUP BY da.calc_date, da.rep_flag
)
/* разворачиваем снимки в одну строку */
SELECT
        a.calc_date                                                      AS report_date,

        MAX(CASE WHEN rep_flag='REP_20250708' THEN cnt_dep         END) AS cnt_dep_20250708,
        MAX(CASE WHEN rep_flag='REP_20250708' THEN sum_balance_rub END) AS sum_balance_rub_20250708,
        MAX(CASE WHEN rep_flag='REP_20250708' THEN sum_out_rub     END) AS sum_out_rub_20250708,
        MAX(CASE WHEN rep_flag='REP_20250708' THEN cnt_no_saldo    END) AS cnt_no_saldo_20250708,

        MAX(CASE WHEN rep_flag='REP_LATEST'  THEN cnt_dep         END)  AS cnt_dep_latest,
        MAX(CASE WHEN rep_flag='REP_LATEST'  THEN sum_balance_rub END)  AS sum_balance_rub_latest,
        MAX(CASE WHEN rep_flag='REP_LATEST'  THEN sum_out_rub     END)  AS sum_out_rub_latest,
        MAX(CASE WHEN rep_flag='REP_LATEST'  THEN cnt_no_saldo    END)  AS cnt_no_saldo_latest
FROM   agg a
GROUP BY a.calc_date
ORDER BY a.calc_date
OPTION (MAXRECURSION 200, RECOMPILE);
