SELECT
    @@SPID AS session_id,

    CAST(
        (user_objects_alloc_page_count
        - user_objects_dealloc_page_count)
        / 128.0 AS decimal(18,2)
    ) AS temp_tables_mb,

    CAST(
        (internal_objects_alloc_page_count
        - internal_objects_dealloc_page_count)
        / 128.0 AS decimal(18,2)
    ) AS internal_objects_mb

FROM tempdb.sys.dm_db_session_space_usage
WHERE session_id = @@SPID;
