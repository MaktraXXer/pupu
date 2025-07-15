USE ALM_TEST;
GO
/*--------------------------------------------------------------------
   mail.usp_fill_balance_metrics_savings
--------------------------------------------------------------------*/
ALTER PROCEDURE mail.usp_fill_balance_metrics_savings
      @DateTo   date = NULL        -- «включительно»
    , @DaysBack int  = 21
AS
BEGIN
    SET NOCOUNT ON;

    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));

    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

;WITH t_ext AS (
    SELECT
          t.dt_rep
        , t.dt_open
        , t.con_id
        , t.out_rub
        , t.rate_con                      AS rate_balance    -- t
        , r.rate                          AS rate_liq        -- r
        , pos.rate_con                    AS rate_pos        -- первая + ставка за +1/+2 сут.

    FROM alm.ALM.vw_balance_rest_all AS t

    /* ставка из DepositContract_Rate */
    LEFT JOIN LIQUIDITY.liq.DepositContract_Rate r
           ON  r.con_id = t.con_id
           AND CASE WHEN t.dt_open = t.dt_rep
                    THEN DATEADD(day,1,t.dt_rep)     -- первый «нулевой» день ULTRA
                    ELSE t.dt_rep
               END BETWEEN r.dt_from AND r.dt_to

    /* ищем положительную ставку из ULTRA в +1 / +2 суток */
    OUTER APPLY (
        SELECT TOP (1) p.rate_con
        FROM   alm.ALM.vw_balance_rest_all p
        WHERE  p.con_id       = t.con_id
          AND  p.dt_rep      IN (DATEADD(day,1,t.dt_rep), DATEADD(day,2,t.dt_rep))
          AND  p.rate_con     > 0
          AND  p.rate_con_src = N'счет ультра,вручную'
        ORDER BY p.dt_rep
    ) pos

    WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
      AND t.section_name = N'Накопительный счёт'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
),

rate_calc AS (
    SELECT *,
           CASE
             /* ---------- r IS NULL ------------------------------------ */
             WHEN rate_liq IS NULL
                  THEN CASE
                          WHEN rate_balance < 0
                               THEN COALESCE(rate_pos, rate_balance)
                          ELSE rate_balance
                       END

             /* ---------- (1) r <0, t >0 ------------------------------- */
             WHEN rate_liq < 0  AND rate_balance > 0
                  THEN rate_balance

             /* ---------- (2) r <0, t <0 ------------------------------- */
             WHEN rate_liq < 0  AND rate_balance < 0
                  THEN COALESCE(rate_pos, rate_balance)

             /* ---------- (3) r ≥0, t ≥0 ------------------------------- */
             WHEN rate_liq >= 0 AND rate_balance >= 0
                  THEN rate_liq

             /* ---------- (4) r >0, t <0 ------------------------------- */
             WHEN rate_liq > 0  AND rate_balance < 0
                  THEN rate_liq

             /* ---------- fallback ------------------------------------- */
             ELSE rate_liq
           END AS rate_use
    FROM t_ext
),

aggr_all AS (
    SELECT dt_rep,
           SUM(out_rub)                                AS out_rub_total,
           NULL                                        AS term_day,
           SUM(rate_use * out_rub) / NULLIF(SUM(out_rub),0) AS rate_con
    FROM rate_calc
    GROUP BY dt_rep
),
aggr_new AS (
    SELECT dt_rep,
           SUM(out_rub)                                AS out_rub_total,
           NULL                                        AS term_day,
           SUM(rate_use * out_rub) / NULLIF(SUM(out_rub),0) AS rate_con
    FROM rate_calc
    WHERE dt_open = dt_rep
    GROUP BY dt_rep
),

cal AS (
    SELECT TOP (DATEDIFF(day,@DateFrom,@DateTo)+1)
           DATEADD(day,ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1,@DateFrom) AS dt_rep
    FROM master..spt_values
),
scopes AS (SELECT 'портфель' AS data_scope UNION ALL SELECT 'новые'),
frame  AS (SELECT c.dt_rep, s.data_scope FROM cal c CROSS JOIN scopes s),
src AS (
    SELECT
          f.dt_rep,
          f.data_scope,
          COALESCE(a.out_rub_total , n.out_rub_total) AS out_rub_total,
          NULL                                         AS term_day,
          COALESCE(a.rate_con      , n.rate_con)       AS rate_con
    FROM frame f
    LEFT JOIN aggr_all a ON a.dt_rep = f.dt_rep AND f.data_scope = 'портфель'
    LEFT JOIN aggr_new n ON n.dt_rep = f.dt_rep AND f.data_scope = 'новые'
)

MERGE mail.balance_metrics_savings AS tgt
USING src                         AS src
  ON  tgt.dt_rep     = src.dt_rep
  AND tgt.data_scope = src.data_scope
WHEN MATCHED THEN
    UPDATE SET tgt.out_rub_total = src.out_rub_total,
               tgt.rate_con      = src.rate_con,
               tgt.load_dttm     = SYSUTCDATETIME()
WHEN NOT MATCHED BY TARGET THEN
    INSERT (dt_rep, data_scope, out_rub_total, term_day, rate_con)
    VALUES (src.dt_rep, src.data_scope, src.out_rub_total,
            src.term_day, src.rate_con);
END;
GO
