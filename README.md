/* ДИАГНОСТИКА ПОДКЛЮЧЕНИЯ */

SELECT
    SYSDATETIME() AS check_time,
    @@SERVERNAME AS server_name,
    DB_NAME() AS current_db,
    @@SPID AS session_id,
    @@TRANCOUNT AS open_transactions;


/* ЗАПОЛНЕННОСТЬ TEMPDB */

EXEC tempdb.sys.sp_executesql N'
SELECT
    CAST(SUM(CAST(size AS bigint))
        / 128.0 AS decimal(18,2)) AS total_mb,

    CAST(SUM(CAST(FILEPROPERTY(name, ''SpaceUsed'') AS bigint))
        / 128.0 AS decimal(18,2)) AS used_mb,

    CAST(SUM(CAST(size AS bigint)
        - FILEPROPERTY(name, ''SpaceUsed''))
        / 128.0 AS decimal(18,2)) AS free_mb,

    CAST(
        100.0 * SUM(CAST(FILEPROPERTY(name, ''SpaceUsed'') AS bigint))
        / NULLIF(SUM(CAST(size AS bigint)), 0)
        AS decimal(9,2)
    ) AS used_pct

FROM sys.database_files
WHERE type = 0;
';


/* НАГРУЗКА СОБСТВЕННОЙ СЕССИИ */

SELECT
    session_id,
    cpu_time / 1000.0 AS cpu_seconds,
    memory_usage * 8.0 / 1024 AS session_memory_mb,
    reads,
    writes,
    logical_reads,
    open_transaction_count
FROM sys.dm_exec_sessions
WHERE session_id = @@SPID;


/* РАЗМЕРЫ НАШИХ ВРЕМЕННЫХ ТАБЛИЦ */

IF OBJECT_ID('tempdb..#rk26p_cli') IS NOT NULL
    EXEC tempdb.sys.sp_spaceused N'#rk26p_cli';

IF OBJECT_ID('tempdb..#rk26w_msg') IS NOT NULL
    EXEC tempdb.sys.sp_spaceused N'#rk26w_msg';

IF OBJECT_ID('tempdb..#rk26w_event') IS NOT NULL
    EXEC tempdb.sys.sp_spaceused N'#rk26w_event';

IF OBJECT_ID('tempdb..#rk26w_windows') IS NOT NULL
    EXEC tempdb.sys.sp_spaceused N'#rk26w_windows';

IF OBJECT_ID('tempdb..#rk26w_dayflags') IS NOT NULL
    EXEC tempdb.sys.sp_spaceused N'#rk26w_dayflags';

IF OBJECT_ID('tempdb..#rk26w_agg') IS NOT NULL
    EXEC tempdb.sys.sp_spaceused N'#rk26w_agg';
