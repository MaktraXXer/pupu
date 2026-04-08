Ты запутался из-за имен constraints.

В SQL Server имя PK_balance_metrics_td и DF_bmtd_load_dttm — это имена объектов в схеме, они должны быть уникальны.
Поэтому если старая таблица [mail].[balance_metrics_td] уже существует с такими constraint names, то создать [mail].[balance_metrics_td_lesha] с теми же именами нельзя.

Что надо сделать:
	1.	либо дать новые имена PK и default constraint для новой таблицы;
	2.	либо вообще не задавать имя constraint вручную, тогда SQL Server сам создаст.

Ниже нормальный вариант для новой таблицы.

USE [ALM_TEST]
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

CREATE TABLE [mail].[balance_metrics_td_lesha](
    [dt_rep] [date] NOT NULL,
     NOT NULL,
    [out_rub_total] [decimal](19, 2) NULL,
    [term_day] [numeric](18, 2) NULL,
    [rate_con] [numeric](18, 6) NULL,
     NOT NULL,
    [deal_term_day] [numeric](18, 2) NULL,
    CONSTRAINT [PK_balance_metrics_td_lesha] PRIMARY KEY CLUSTERED
    (
        [dt_rep] ASC,
        [data_scope] ASC
    ) WITH (
        PAD_INDEX = OFF,
        STATISTICS_NORECOMPUTE = OFF,
        IGNORE_DUP_KEY = OFF,
        ALLOW_ROW_LOCKS = ON,
        ALLOW_PAGE_LOCKS = ON,
        OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF
    ) ON [PRIMARY]
) ON [PRIMARY]
GO

ALTER TABLE [mail].[balance_metrics_td_lesha]
ADD CONSTRAINT [DF_balance_metrics_td_lesha_load_dttm]
DEFAULT (sysutcdatetime()) FOR [load_dttm]
GO

Теперь процу тоже надо переписать, чтобы она писала в новую таблицу:

USE [ALM_TEST]
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

ALTER PROCEDURE [mail].[usp_fill_balance_metrics_depo]
      @DateTo   date = NULL
    , @DaysBack int  = 21
AS
BEGIN
    SET NOCOUNT ON;

    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day, -2, CAST(GETDATE() AS date));

    DECLARE @DateFrom date = DATEADD(day, -@DaysBack + 1, @DateTo);

    ;WITH aggr_all AS (
        SELECT
              t.dt_rep
            , SUM(t.out_rub) AS out_rub_total
            , SUM(DATEDIFF(day, t.dt_rep, t.dt_close) * t.out_rub)
                / NULLIF(SUM(t.out_rub), 0) AS term_day
            , SUM(CASE WHEN t.rate_con IS NOT NULL THEN t.rate_con * t.out_rub ELSE 0 END)
                / NULLIF(SUM(CASE WHEN t.rate_con IS NOT NULL THEN t.out_rub ELSE 0 END), 0) AS rate_con
            , CAST(NULL AS numeric(18,2)) AS deal_term_day
        FROM alm.[ALM].[vw_balance_rest_all] t WITH (NOLOCK)
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.section_name = N'Срочные'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
        GROUP BY t.dt_rep
    ),
    aggr_new AS (
        SELECT
              t.dt_rep
            , SUM(t.out_rub) AS out_rub_total
            , SUM(DATEDIFF(day, t.dt_rep, t.dt_close) * t.out_rub)
                / NULLIF(SUM(t.out_rub), 0) AS term_day
            , SUM(CASE WHEN t.rate_con IS NOT NULL THEN t.rate_con * t.out_rub ELSE 0 END)
                / NULLIF(SUM(CASE WHEN t.rate_con IS NOT NULL THEN t.out_rub ELSE 0 END), 0) AS rate_con
            , SUM(DATEDIFF(day, t.dt_open, t.dt_close) * t.out_rub)
                / NULLIF(SUM(t.out_rub), 0) AS deal_term_day
        FROM alm.[ALM].[vw_balance_rest_all] t WITH (NOLOCK)
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.dt_open = t.dt_rep
          AND t.section_name = N'Срочные'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
        GROUP BY t.dt_rep
    ),
    cal AS (
        SELECT TOP (DATEDIFF(day, @DateFrom, @DateTo) + 1)
               DATEADD(day, ROW_NUMBER() OVER (ORDER BY (SELECT 1)) - 1, @DateFrom) AS dt_rep
        FROM master..spt_values
    ),
    scopes AS (
        SELECT N'портфель' AS data_scope
        UNION ALL
        SELECT N'новые'
    ),
    frame AS (
        SELECT c.dt_rep, s.data_scope
        FROM cal c
        CROSS JOIN scopes s
    ),
    src AS (
        SELECT
              f.dt_rep
            , f.data_scope
            , COALESCE(a.out_rub_total, n.out_rub_total) AS out_rub_total
            , COALESCE(a.term_day, n.term_day) AS term_day
            , COALESCE(a.rate_con, n.rate_con) AS rate_con
            , COALESCE(a.deal_term_day, n.deal_term_day) AS deal_term_day
        FROM frame f
        LEFT JOIN aggr_all a
            ON a.dt_rep = f.dt_rep
           AND f.data_scope = N'портфель'
        LEFT JOIN aggr_new n
            ON n.dt_rep = f.dt_rep
           AND f.data_scope = N'новые'
    )
    MERGE mail.balance_metrics_td_lesha AS tgt
    USING src AS src
      ON tgt.dt_rep = src.dt_rep
     AND tgt.data_scope = src.data_scope
    WHEN MATCHED THEN
        UPDATE SET
              tgt.out_rub_total = src.out_rub_total
            , tgt.term_day = src.term_day
            , tgt.rate_con = src.rate_con
            , tgt.deal_term_day = src.deal_term_day
            , tgt.load_dttm = SYSUTCDATETIME()
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (
              dt_rep
            , data_scope
            , out_rub_total
            , term_day
            , rate_con
            , deal_term_day
        )
        VALUES (
              src.dt_rep
            , src.data_scope
            , src.out_rub_total
            , src.term_day
            , src.rate_con
            , src.deal_term_day
        );
END
GO

Если хочешь быстро проверить, не создались ли уже какие-то кривые constraints, выполни:

SELECT
    t.name AS table_name,
    kc.name AS constraint_name,
    kc.type_desc
FROM sys.key_constraints kc
JOIN sys.tables t
    ON kc.parent_object_id = t.object_id
WHERE kc.name IN ('PK_balance_metrics_td', 'PK_balance_metrics_td_lesha');

SELECT
    t.name AS table_name,
    dc.name AS constraint_name
FROM sys.default_constraints dc
JOIN sys.tables t
    ON dc.parent_object_id = t.object_id
WHERE dc.name IN ('DF_bmtd_load_dttm', 'DF_balance_metrics_td_lesha_load_dttm');

Если нужно, я могу сразу дать тебе еще и блок:
IF OBJECT_ID(...) IS NOT NULL DROP TABLE ...
плюс безопасный recreate новой таблицы.
