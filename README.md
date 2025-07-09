/* ежедневная витрина 16-июн-2025 … 07-июл-2025 (две выборки dep)  */
;WITH
------------------------------------------------------------------
-- актуальная дата отчёта ----------------------------------------
latest_rep AS (
    SELECT MAX(DT_REP) AS DT_REP
    FROM  [LIQUIDITY].[liq].[DepositInterestsRate] WITH (NOLOCK)
),
------------------------------------------------------------------
-- календарь -----------------------------------------------------
cal AS (
    SELECT CAST('2025-06-16' AS date) AS calc_date
    UNION ALL
    SELECT DATEADD(day,1,calc_date)
    FROM   cal
    WHERE  calc_date < '2025-07-07'
),
------------------------------------------------------------------
-- депозиты с нужными фильтрами ----------------------------------
dep_filtered AS (
    SELECT  d.CON_ID,
            d.BALANCE_RUB,
            CASE WHEN d.DT_REP = '2025-07-08' THEN 'REP_20250708'
                 ELSE 'REP_LATEST' END AS rep_flag
    FROM   [LIQUIDITY].[liq].[DepositInterestsRate] d WITH (NOLOCK)
    CROSS  JOIN latest_rep lr            -- фиксируем latest единожды
    WHERE  d.CLI_SUBTYPE = N'ФЛ'
      AND  d.isfloat    <> 1
      AND  d.CUR        =  N'RUR'
      AND  d.DT_REP     IN ('2025-07-08', lr.DT_REP)
),
------------------------------------------------------------------
-- сальдо, действующее на каждую дату ----------------------------
saldo_on_date AS (
    SELECT  s.CON_ID,
            c.calc_date,
            s.OUT_RUB
    FROM   [LIQUIDITY].[liq].[DepositContract_Saldo] s WITH (NOLOCK)
    JOIN   cal c
           ON c.calc_date BETWEEN s.DT_FROM AND s.DT_TO
),
------------------------------------------------------------------
-- агрегаты ------------------------------------------------------
agg AS (
    SELECT  c.calc_date,
            d.rep_flag,
            COUNT(DISTINCT d.CON_ID)                                    AS cnt_dep,
            SUM(d.BALANCE_RUB)                                          AS sum_balance_rub,
            SUM(ISNULL(s.OUT_RUB,0))                                    AS sum_out_rub,
            SUM(CASE WHEN s.CON_ID IS NULL THEN 1 ELSE 0 END)           AS cnt_no_saldo
    FROM      cal              c
    CROSS JOIN dep_filtered     d        -- оцениваем каждый договор на каждую дату
    LEFT JOIN saldo_on_date     s
           ON s.CON_ID    = d.CON_ID
          AND s.calc_date = c.calc_date
    GROUP BY c.calc_date, d.rep_flag
)
------------------------------------------------------------------
-- итог ----------------------------------------------------------
SELECT
        a.calc_date                                                       AS report_date,

        MAX(CASE WHEN rep_flag='REP_20250708' THEN cnt_dep         END)   AS cnt_dep_20250708,
        MAX(CASE WHEN rep_flag='REP_20250708' THEN sum_balance_rub END)   AS sum_balance_rub_20250708,
        MAX(CASE WHEN rep_flag='REP_20250708' THEN sum_out_rub     END)   AS sum_out_rub_20250708,
        MAX(CASE WHEN rep_flag='REP_20250708' THEN cnt_no_saldo    END)   AS cnt_no_saldo_20250708,

        MAX(CASE WHEN rep_flag='REP_LATEST'    THEN cnt_dep         END)  AS cnt_dep_latest,
        MAX(CASE WHEN rep_flag='REP_LATEST'    THEN sum_balance_rub END)  AS sum_balance_rub_latest,
        MAX(CASE WHEN rep_flag='REP_LATEST'    THEN sum_out_rub     END)  AS sum_out_rub_latest,
        MAX(CASE WHEN rep_flag='REP_LATEST'    THEN cnt_no_saldo    END)  AS cnt_no_saldo_latest
FROM   agg a
GROUP BY a.calc_date
ORDER BY a.calc_date
OPTION (MAXRECURSION 1000, RECOMPILE);
