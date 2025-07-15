USE ALM_TEST;
GO
/*--------------------------------------------------------------------
   mail.usp_fill_balance_metrics_savings
   (быстрая версия без коррелированного OUTER APPLY)
--------------------------------------------------------------------*/
ALTER PROCEDURE mail.usp_fill_balance_metrics_savings
      @DateTo   date = NULL        -- «включительно»
    , @DaysBack int  = 21
AS
BEGIN
    SET NOCOUNT ON;

    /* --------------------------------------------------------------
       1. Диапазоны дат
    ----------------------------------------------------------------*/
    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));

    DECLARE @DateFrom      date = DATEADD(day,-@DaysBack+1,@DateTo);
    DECLARE @DateFromWide  date = DATEADD(day,-2,@DateFrom);   -- «-2»  для look-ahead
    DECLARE @DateToWide    date = DATEADD(day, 2,@DateTo);     -- «+2»  для look-ahead

    /* --------------------------------------------------------------
       2. Читаем источник ОДИН раз и кладём во временную таблицу
    ----------------------------------------------------------------*/
    IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;

    SELECT
          t.dt_rep,
          t.dt_open,
          t.con_id,
          t.out_rub,
          t.rate_con           AS rate_balance,        -- ставка из баланса (t)
          t.rate_con_src,
          r.rate               AS rate_liq             -- ставка из DepositContract_Rate
    INTO   #bal
    FROM   alm.ALM.vw_balance_rest_all AS t
    LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
           ON  r.con_id = t.con_id
           AND CASE WHEN t.dt_open = t.dt_rep
                    THEN DATEADD(day,1,t.dt_rep)       -- первый «нулевой» день ULTRA
                    ELSE t.dt_rep
               END BETWEEN r.dt_from AND r.dt_to
    WHERE  t.dt_rep BETWEEN @DateFromWide AND @DateToWide
      AND  t.section_name = N'Накопительный счёт'
      AND  t.block_name   = N'Привлечение ФЛ'
      AND  t.od_flag      = 1
      AND  t.cur          = '810';

    /* быстрый упорядоченный доступ внутри оконной функции */
    CREATE CLUSTERED INDEX IX_bal ON #bal (con_id, dt_rep);

    /* --------------------------------------------------------------
       3. Добавляем rate_pos через ОДНУ оконную функцию
    ----------------------------------------------------------------*/
    ;WITH bal_pos AS (
        SELECT *,
               /* первая положительная ставка ULTRA на +1 / +2 день */
               MIN(CASE WHEN rate_balance > 0
                         AND rate_con_src = N'счет ультра,вручную'
                        THEN rate_balance END)
                   OVER (PARTITION BY con_id
                         ORDER BY dt_rep
                         ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING) AS rate_pos
        FROM   #bal
        WHERE  dt_rep BETWEEN @DateFrom AND @DateTo   -- итоговый рабочий диапазон
    ),

    /* --------------------------------------------------------------
       4. Логика выбора rate_use (без изменений правил 1,3,4)
    ----------------------------------------------------------------*/
    rate_calc AS (
        SELECT *,
               CASE
                 /* r IS NULL */
                 WHEN rate_liq IS NULL
                      THEN CASE
                               WHEN rate_balance < 0
                                    THEN COALESCE(rate_pos, rate_balance)
                               ELSE rate_balance
                           END

                 /* (1) r<0 , t>0 */
                 WHEN rate_liq < 0  AND rate_balance > 0
                      THEN rate_balance

                 /* (2) r<0 , t<0 */
                 WHEN rate_liq < 0  AND rate_balance < 0
                      THEN COALESCE(rate_pos, rate_balance)

                 /* (3) r≥0 , t≥0 */
                 WHEN rate_liq >= 0 AND rate_balance >= 0
                      THEN rate_liq

                 /* (4) r>0 , t<0 */
                 WHEN rate_liq > 0  AND rate_balance < 0
                      THEN rate_liq

                 ELSE rate_liq
               END AS rate_use
        FROM bal_pos
    ),

    /* -------------------------------------------------------------- */
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

    /* --------------------------------------------------------------
       5. MERGE в целевую таблицу
    ----------------------------------------------------------------*/
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
