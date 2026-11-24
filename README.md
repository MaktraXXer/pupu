USE ALM_TEST;
GO

SET XACT_ABORT ON;

BEGIN TRANSACTION;

IF COL_LENGTH('alm_report.depo_fl_client_monthly_stats', 'is_push_sent') IS NULL
    ALTER TABLE [alm_report].[depo_fl_client_monthly_stats]
    ADD is_push_sent bit NOT NULL DEFAULT (0);

IF COL_LENGTH('alm_report.depo_fl_client_monthly_stats', 'exit_promo15_td_sum') IS NULL
    ALTER TABLE [alm_report].[depo_fl_client_monthly_stats]
    ADD exit_promo15_td_sum decimal(38,6) NOT NULL DEFAULT (0);

IF COL_LENGTH('alm_report.depo_fl_client_monthly_stats', 'opened_promo15') IS NULL
    ALTER TABLE [alm_report].[depo_fl_client_monthly_stats]
    ADD opened_promo15 decimal(38,6) NOT NULL DEFAULT (0);

TRUNCATE TABLE [alm_report].[depo_fl_client_monthly_stats];

COMMIT TRANSACTION;
GO
