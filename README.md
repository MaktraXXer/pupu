ALTER PROCEDURE [mail].[usp_fill_balance_metrics_depo_only]
      @DateTo   date = NULL        -- включительно
    , @DaysBack int  = 21          -- длина окна
AS
BEGIN
    SET NOCOUNT ON;

    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));
    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    ;WITH
    -- 1) Фиксируем список маркет-продуктов по ИМЕНИ (как в сверке)
    mp_names AS (
        SELECT DISTINCT LTRIM(RTRIM(p.PROD_NAME)) AS PROD_NAME
        FROM ALM_TEST.markets.prod_term_rates p WITH (NOLOCK)
    ),
    -- 2) База: только "Срочные / Привлечение ФЛ / cur=810 / активные", только маркет-продукты, и только положительный out_rub
    base AS (
        SELECT
              t.*
            , DATEDIFF(day, t.dt_open, t.dt_close) AS deal_term_days
        FROM  alm.[ALM].[vw_balance_rest_all] t WITH (NOLOCK)
        JOIN  mp_names mp
              ON LTRIM(RTRIM(t.prod_name_res)) = mp.PROD_NAME
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.section_name = N'Срочные'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
          AND t.out_rub IS NOT NULL
          AND t.out_rub > 0
    ),
    -- 3) Портфель (все действующие на дату): надбавка подключается, но строки не отваливаются
    aggr_all AS (
        SELECT
              b.dt_rep
            , SUM(b.out_rub) AS out_rub_total
            , SUM(DATEDIFF(day,b.dt_rep,b.dt_close) * b.out_rub) / NULLIF(SUM(b.out_rub),0) AS term_day
            , SUM(CASE WHEN b.rate_con IS NOT NULL
                       THEN (b.rate_con + COALESCE(ap.PERCRATE,0)) * b.out_rub
                       ELSE 0 END)
              / NULLIF(SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub ELSE 0 END),0) AS rate_con
            , CAST(NULL AS numeric(18,2)) AS deal_term_day
        FROM base b
        OUTER APPLY (
            SELECT TOP (1) p.PERCRATE
            FROM ALM_TEST.markets.prod_term_rates p WITH (NOLOCK)
            WHERE p.PROD_ID = b.prod_id
              AND b.deal_term_days BETWEEN p.TERMDAYS_FROM AND p.TERMDAYS_TO
              AND b.dt_open BETWEEN p.DT_FROM AND p.DT_TO     -- при необходимости можно сменить на b.dt_rep
            ORDER BY p.DT_FROM DESC, p.DT_TO DESC, p.TERMDAYS_FROM DESC
        ) ap
        GROUP BY b.dt_rep
    ),
    -- 4) Новые сделки на дату открытия
    aggr_new AS (
        SELECT
              b.dt_rep
            , SUM(b.out_rub) AS out_rub_total
            , SUM(DATEDIFF(day,b.dt_rep,b.dt_close) * b.out_rub) / NULLIF(SUM(b.out_rub),0) AS term_day
            , SUM(CASE WHEN b.rate_con IS NOT NULL
                       THEN (b.rate_con + COALESCE(ap.PERCRATE,0)) * b.out_rub
                       ELSE 0 END)
              / NULLIF(SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub ELSE 0 END),0) AS rate_con
            , SUM(DATEDIFF(day,b.dt_open,b.dt_close) * b.out_rub) / NULLIF(SUM(b.out_rub),0) AS deal_term_day
        FROM base b
        OUTER APPLY (
            SELECT TOP (1) p.PERCRATE
            FROM ALM_TEST.markets.prod_term_rates p WITH (NOLOCK)
            WHERE p.PROD_ID = b.prod_id
              AND b.deal_term_days BETWEEN p.TERMDAYS_FROM AND p.TERMDAYS_TO
              AND b.dt_open BETWEEN p.DT_FROM AND p.DT_TO     -- при необходимости можно сменить на b.dt_rep
            ORDER BY p.DT_FROM DESC, p.DT_TO DESC, p.TERMDAYS_FROM DESC
        ) ap
        WHERE b.dt_open = b.dt_rep
        GROUP BY b.dt_rep
    ),
    -- 5) Календарная рамка
    cal AS (
        SELECT TOP (DATEDIFF(day,@DateFrom,@DateTo)+1)
               DATEADD(day,ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1,@DateFrom) AS dt_rep
        FROM master..spt_values
    ),
    scopes AS (SELECT N'портфель' AS data_scope UNION ALL SELECT N'новые'),
    frame AS (SELECT c.dt_rep, s.data_scope FROM cal c CROSS JOIN scopes s),
    -- 6) Объединяем источники для MERGE
    src AS (
        SELECT
              f.dt_rep
            , f.data_scope
            , CASE WHEN f.data_scope=N'портфель' THEN a.out_rub_total ELSE n.out_rub_total END AS out_rub_total
            , CASE WHEN f.data_scope=N'портфель' THEN a.term_day      ELSE n.term_day      END AS term_day
            , CASE WHEN f.data_scope=N'портфель' THEN a.rate_con      ELSE n.rate_con      END AS rate_con
            , CASE WHEN f.data_scope=N'портфель' THEN a.deal_term_day ELSE n.deal_term_day END AS deal_term_day
        FROM frame f
        LEFT JOIN aggr_all a ON a.dt_rep = f.dt_rep AND f.data_scope = N'портфель'
        LEFT JOIN aggr_new n ON n.dt_rep = f.dt_rep AND f.data_scope = N'новые'
    )

    MERGE mail.balance_metrics_td_only AS tgt
    USING src AS src
      ON  tgt.dt_rep     = src.dt_rep
      AND tgt.data_scope = src.data_scope
    WHEN MATCHED THEN
        UPDATE SET tgt.out_rub_total = src.out_rub_total,
                   tgt.term_day      = src.term_day,
                   tgt.rate_con      = src.rate_con,
                   tgt.deal_term_day = src.deal_term_day,
                   tgt.load_dttm     = SYSUTCDATETIME()
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (dt_rep, data_scope, out_rub_total, term_day, rate_con, deal_term_day)
        VALUES (src.dt_rep, src.data_scope, src.out_rub_total, src.term_day, src.rate_con, src.deal_term_day);
END
GO
