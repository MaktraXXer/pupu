USE [ALM_TEST]
GO

SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE TABLE [mail].[balance_metrics_td_lesha](
    [dt_rep]         [date]           NOT NULL,
    [data_scope]     [nvarchar](10)   NOT NULL,
    [out_rub_total]  [decimal](19, 2) NULL,
    [term_day]       [numeric](18, 2) NULL,
    [rate_con]       [numeric](18, 6) NULL,
    [load_dttm]      [datetime2](3)   NOT NULL,
    [deal_term_day]  [numeric](18, 2) NULL,
    CONSTRAINT [PK_balance_metrics_td_lesha] PRIMARY KEY CLUSTERED 
    (
        [dt_rep] ASC,
        [data_scope] ASC
    ) WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, 
            IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, 
            ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
) ON [PRIMARY]
GO

ALTER TABLE [mail].[balance_metrics_td_lesha] 
    ADD CONSTRAINT [DF_bmtd_lesha_load_dttm] 
    DEFAULT (sysutcdatetime()) FOR [load_dttm]
GO
