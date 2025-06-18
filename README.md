-- Удаляем constraint на дефолтное значение, если он остался
IF OBJECT_ID('DF_vint_load', 'D') IS NOT NULL
    ALTER TABLE alm_test.dbo.fu_vintage_results
    DROP CONSTRAINT DF_vint_load;

-- Удаляем индексы, если они остались
IF EXISTS (SELECT * FROM sys.indexes WHERE name = 'IX_fu_vint_rep_gen' AND object_id = OBJECT_ID('alm_test.dbo.fu_vintage_results'))
    DROP INDEX IX_fu_vint_rep_gen ON alm_test.dbo.fu_vintage_results;

IF EXISTS (SELECT * FROM sys.indexes WHERE name = 'IX_fu_vint_vint_had' AND object_id = OBJECT_ID('alm_test.dbo.fu_vintage_results'))
    DROP INDEX IX_fu_vint_vint_had ON alm_test.dbo.fu_vintage_results;

-- Удаляем таблицу, если она осталась
IF OBJECT_ID('alm_test.dbo.fu_vintage_results','U') IS NOT NULL
    DROP TABLE alm_test.dbo.fu_vintage_results;
