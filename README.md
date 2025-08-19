Отлично, сделал два отдельных хранилища и две процедуры-загрузчики:
A) «исключаем продукты из списка» (без учёта PERCRATE),
B) «берём только продукты из списка» (с добавлением PERCRATE к rate_con по правилам сопоставления).

Ничего лишнего — структура и логика максимально близки к вашей, фильтры и расчёты встроены прямо в агрегации без тяжёлых JOIN’ов. Если хотите — можно оставить имена/схемы как есть, я задал очевидные.

⸻

1) Таблица под вариант A (исключаем PROD_ID из списка)

USE [ALM_TEST];
GO

SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

CREATE TABLE [mail].[balance_metrics_td_excl](
    [dt_rep]        date            NOT NULL,
    [data_scope]    nvarchar(10)    NOT NULL,
    [out_rub_total] decimal(19,2)   NULL,
    [term_day]      numeric(18,2)   NULL,
    [rate_con]      numeric(18,6)   NULL,
    [load_dttm]     datetime2(3)    NOT NULL CONSTRAINT [DF_bmtd_excl_load_dttm] DEFAULT (sysutcdatetime()),
    [deal_term_day] numeric(18,2)   NULL,
    CONSTRAINT [PK_balance_metrics_td_excl] PRIMARY KEY CLUSTERED ([dt_rep],[data_scope])
) ON [PRIMARY];
GO

2) Таблица под вариант B (берём только PROD_ID из списка и учитываем PERCRATE)

USE [ALM_TEST];
GO

SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

CREATE TABLE [mail].[balance_metrics_td_only](
    [dt_rep]        date            NOT NULL,
    [data_scope]    nvarchar(10)    NOT NULL,
    [out_rub_total] decimal(19,2)   NULL,
    [term_day]      numeric(18,2)   NULL,
    [rate_con]      numeric(18,6)   NULL,
    [load_dttm]     datetime2(3)    NOT NULL CONSTRAINT [DF_bmtd_only_load_dttm] DEFAULT (sysutcdatetime()),
    [deal_term_day] numeric(18,2)   NULL,
    CONSTRAINT [PK_balance_metrics_td_only] PRIMARY KEY CLUSTERED ([dt_rep],[data_scope])
) ON [PRIMARY];
GO


⸻

3) Процедура A: «Исключить продукты из списка»

— Вся логика как у вас, но добавлен фильтр NOT EXISTS (...) по списку markets.prod_term_rates (только по PROD_ID).
— Ставка rate_con считается без изменений.

USE [ALM_TEST];
GO

SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

CREATE OR ALTER PROCEDURE [mail].[usp_fill_balance_metrics_depo_excl]
      @DateTo   date = NULL        -- включительно
    , @DaysBack int  = 21          -- длина окна
AS
BEGIN
    SET NOCOUNT ON;

    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));
    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    ;WITH aggr_all AS (
        SELECT
              t.dt_rep
            , SUM(t.out_rub) AS out_rub_total
            , SUM(DATEDIFF(day,t.dt_rep,t.dt_close)*t.out_rub) / NULLIF(SUM(t.out_rub),0) AS term_day
            , SUM(CASE WHEN t.rate_con IS NOT NULL THEN t.rate_con * t.out_rub ELSE 0 END)
              / NULLIF(SUM(CASE WHEN t.rate_con IS NOT NULL THEN t.out_rub ELSE 0 END),0) AS rate_con
            , CAST(NULL AS numeric(18,2)) AS deal_term_day
        FROM  alm.[ALM].[vw_balance_rest_all] t
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.section_name = N'Срочные'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
          AND NOT EXISTS (
                SELECT 1
                FROM [markets].[prod_term_rates] p
                WHERE p.PROD_ID = t.prod_id
          )
        GROUP BY t.dt_rep
    ),
    aggr_new AS (
        SELECT
              t.dt_rep
            , SUM(t.out_rub) AS out_rub_total
            , SUM(DATEDIFF(day,t.dt_rep,t.dt_close)*t.out_rub) / NULLIF(SUM(t.out_rub),0) AS term_day
            , SUM(CASE WHEN t.rate_con IS NOT NULL THEN t.rate_con * t.out_rub ELSE 0 END)
              / NULLIF(SUM(CASE WHEN t.rate_con IS NOT NULL THEN t.out_rub ELSE 0 END),0) AS rate_con
            , SUM(DATEDIFF(day,t.dt_open,t.dt_close)*t.out_rub) / NULLIF(SUM(t.out_rub),0) AS deal_term_day
        FROM  alm.[ALM].[vw_balance_rest_all] t
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.dt_open = t.dt_rep
          AND t.section_name = N'Срочные'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
          AND NOT EXISTS (
                SELECT 1
                FROM [markets].[prod_term_rates] p
                WHERE p.PROD_ID = t.prod_id
          )
        GROUP BY t.dt_rep
    ),
    cal AS (
        SELECT TOP (DATEDIFF(day,@DateFrom,@DateTo)+1)
               DATEADD(day,ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1,@DateFrom) AS dt_rep
        FROM master..spt_values
    ),
    scopes AS (SELECT N'портфель' AS data_scope UNION ALL SELECT N'новые'),
    frame AS (SELECT c.dt_rep, s.data_scope FROM cal c CROSS JOIN scopes s),
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

    MERGE mail.balance_metrics_td_excl AS tgt
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


⸻

4) Процедура B: «Только продукты из списка + PERCRATE»

— Берём только депозиты, для которых находится запись в markets.prod_term_rates по правилам:
p.PROD_ID = t.prod_id и t.dt_open BETWEEN p.DT_FROM AND p.DT_TO
и DATEDIFF(day,t.dt_open,t.dt_close) BETWEEN p.TERMDAYS_FROM AND p.TERMDAYS_TO.
— Чтоб не раздувать строки и не плодить дубликаты, берём TOP (1) подходящую запись через CROSS APPLY (самый «свежий» по DT_FROM).
— В средневзвешенной ставке используем rate_con + PERCRATE (если rate_con IS NULL, позицию не учитываем — как у вас).

USE [ALM_TEST];
GO

SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

CREATE OR ALTER PROCEDURE [mail].[usp_fill_balance_metrics_depo_only]
      @DateTo   date = NULL        -- включительно
    , @DaysBack int  = 21          -- длина окна
AS
BEGIN
    SET NOCOUNT ON;

    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));
    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    /* общий фильтр и сопоставление правил через CROSS APPLY */
    ;WITH base AS (
        SELECT
              t.*
            , DATEDIFF(day, t.dt_open, t.dt_close) AS deal_term_days
        FROM  alm.[ALM].[vw_balance_rest_all] t
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.section_name = N'Срочные'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
    ),
    /* портфель */
    aggr_all AS (
        SELECT
              b.dt_rep
            , SUM(b.out_rub) AS out_rub_total
            , SUM(DATEDIFF(day,b.dt_rep,b.dt_close)*b.out_rub) / NULLIF(SUM(b.out_rub),0) AS term_day
            , SUM(CASE WHEN b.rate_con IS NOT NULL THEN (b.rate_con + COALESCE(ap.PERCRATE,0)) * b.out_rub ELSE 0 END)
              / NULLIF(SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub ELSE 0 END),0) AS rate_con
            , CAST(NULL AS numeric(18,2)) AS deal_term_day
        FROM base b
        CROSS APPLY (
            SELECT TOP (1) p.PERCRATE
            FROM [markets].[prod_term_rates] p
            WHERE p.PROD_ID = b.prod_id
              AND b.dt_open BETWEEN p.DT_FROM AND p.DT_TO
              AND b.deal_term_days BETWEEN p.TERMDAYS_FROM AND p.TERMDAYS_TO
            ORDER BY p.DT_FROM DESC
        ) ap
        GROUP BY b.dt_rep
    ),
    /* новые сделки */
    aggr_new AS (
        SELECT
              b.dt_rep
            , SUM(b.out_rub) AS out_rub_total
            , SUM(DATEDIFF(day,b.dt_rep,b.dt_close)*b.out_rub) / NULLIF(SUM(b.out_rub),0) AS term_day
            , SUM(CASE WHEN b.rate_con IS NOT NULL THEN (b.rate_con + COALESCE(ap.PERCRATE,0)) * b.out_rub ELSE 0 END)
              / NULLIF(SUM(CASE WHEN b.rate_con IS NOT NULL THEN b.out_rub ELSE 0 END),0) AS rate_con
            , SUM(DATEDIFF(day,b.dt_open,b.dt_close)*b.out_rub) / NULLIF(SUM(b.out_rub),0) AS deal_term_day
        FROM base b
        CROSS APPLY (
            SELECT TOP (1) p.PERCRATE
            FROM [markets].[prod_term_rates] p
            WHERE p.PROD_ID = b.prod_id
              AND b.dt_open BETWEEN p.DT_FROM AND p.DT_TO
              AND b.deal_term_days BETWEEN p.TERMDAYS_FROM AND p.TERMDAYS_TO
            ORDER BY p.DT_FROM DESC
        ) ap
        WHERE b.dt_open = b.dt_rep
        GROUP BY b.dt_rep
    ),
    cal AS (
        SELECT TOP (DATEDIFF(day,@DateFrom,@DateTo)+1)
               DATEADD(day,ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1,@DateFrom) AS dt_rep
        FROM master..spt_values
    ),
    scopes AS (SELECT N'портфель' AS data_scope UNION ALL SELECT N'новые'),
    frame AS (SELECT c.dt_rep, s.data_scope FROM cal c CROSS JOIN scopes s),
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


⸻

Если нужно, переименую объекты под ваши конвенции или добавлю индексы (например, по markets.prod_term_rates (PROD_ID, DT_FROM, DT_TO, TERMDAYS_FROM, TERMDAYS_TO) для ускорения CROSS APPLY).
