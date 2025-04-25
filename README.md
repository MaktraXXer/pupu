USE ALM_TEST;
GO

/* если таблица-копия уже есть, сначала удаляем */
IF OBJECT_ID('WORK.prolongationAnalysisResult_ISOPTION_backup', 'U') IS NOT NULL
    DROP TABLE WORK.prolongationAnalysisResult_ISOPTION_backup;

/* создаём копию структуры + данных */
SELECT *
INTO   WORK.prolongationAnalysisResult_ISOPTION_backup     -- новое имя
FROM   WORK.prolongationAnalysisResult_ISOPTION;           -- исходная
GO
