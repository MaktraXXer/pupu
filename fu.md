USE [ALM_TEST];
GO

IF SCHEMA_ID('WORK') IS NULL
BEGIN
    EXEC('CREATE SCHEMA [WORK]');
END;
GO

IF OBJECT_ID('[WORK].[TransfertRates]', 'U') IS NOT NULL
BEGIN
    DROP TABLE [WORK].[TransfertRates];
END;
GO

CREATE TABLE [WORK].[TransfertRates](
    [id] [int] IDENTITY(1,1) NOT NULL,
    [RateDate] [date] NOT NULL,
    [IsSettled] [smallint] NOT NULL,
    [OrganizationID] [int] NOT NULL,
    [Type] [varchar](50) NOT NULL,
    [Term] [int] NOT NULL,
    [Currency] [varchar](3) NOT NULL,
    [Rate] [float] NOT NULL,
    [UploadedOn] [datetime] NOT NULL,
    [UploadedBy] [varchar](200) NOT NULL
) ON [PRIMARY];
GO

SET IDENTITY_INSERT [WORK].[TransfertRates] ON;

INSERT INTO [WORK].[TransfertRates] (
    [id],
    [RateDate],
    [IsSettled],
    [OrganizationID],
    [Type],
    [Term],
    [Currency],
    [Rate],
    [UploadedOn],
    [UploadedBy]
)
SELECT
    [id],
    [RateDate],
    [IsSettled],
    [OrganizationID],
    [Type],
    [Term],
    [Currency],
    [Rate],
    [UploadedOn],
    [UploadedBy]
FROM [ALM].[dbo].[TransfertRates];

SET IDENTITY_INSERT [WORK].[TransfertRates] OFF;
GO

SELECT COUNT(*) AS rows_copied
FROM [WORK].[TransfertRates];
