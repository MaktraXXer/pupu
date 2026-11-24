USE ALM;

SELECT
    c.column_id,
    c.name AS column_name,
    TYPE_NAME(c.user_type_id) AS data_type,
    c.precision,
    c.scale,
    c.is_nullable
FROM sys.columns c
WHERE c.object_id = OBJECT_ID(N'ehd.VW_transfers_FL_det')
ORDER BY c.column_id;
