USE tempdb;

SELECT
    SUM(CAST(size AS bigint)) / 128.0 AS total_mb,
    SUM(CAST(FILEPROPERTY(name, 'SpaceUsed') AS bigint))
        / 128.0 AS used_mb,
    SUM(
        CAST(size AS bigint)
        - FILEPROPERTY(name, 'SpaceUsed')
    ) / 128.0 AS free_mb
FROM sys.database_files
WHERE type = 0;

USE ALM;
