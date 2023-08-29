# Query to get rowcount and storage utilization

```sql
--
-- Query to get rowcount and storage utilization
--
SELECT s.NAME                                                                                AS SchemaName,
       t.NAME                                                                                AS TableName,
       p.rows                                                                                AS RowCounts,
       Cast(Round(( Sum(a.used_pages) / 128.00 ), 2) AS NUMERIC(36, 2))                      AS Used_MB,
       Cast(Round(( Sum(a.total_pages) - Sum(a.used_pages) ) / 128.00, 2) AS NUMERIC(36, 2)) AS Unused_MB,
       Cast(Round(( Sum(a.total_pages) / 128.00 ), 2) AS NUMERIC(36, 2))                     AS Total_MB
FROM   sys.tables t
       INNER JOIN sys.indexes i ON t.OBJECT_ID = i.object_id
       INNER JOIN sys.partitions p ON i.object_id = p.OBJECT_ID
           AND i.index_id = p.index_id
       INNER JOIN sys.allocation_units a ON p.partition_id = a.container_id
       INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
GROUP  BY t.NAME, s.NAME, p.Rows
ORDER  BY s.NAME, t.NAME
 
-- WITH LOB
SELECT DB_NAME()                                                                             AS DatabaseName,
       s.NAME                                                                                AS SchemaName,
       t.NAME                                                                                AS TableName,
       i.name                                                                                AS IndexName,
       p.rows                                                                                AS RowCounts,
       Cast(Round(( Sum(a.used_pages) / 128.00 ), 2) AS NUMERIC(36, 2))                      AS Used_MB,
       Cast(Round(( Sum(a.total_pages) - Sum(a.used_pages) ) / 128.00, 2) AS NUMERIC(36, 2)) AS Unused_MB,
       Cast(Round(( Sum(a.total_pages) / 128.00 ), 2) AS NUMERIC(36, 2))                     AS Total_MB,
       SUM(in_row_data_page_count)                                                           AS DATA_PAGE_COUNT,
       SUM(in_row_used_page_count)                                                           AS DATA_PAGE_USED,
       SUM(lob_used_page_count)                                                              AS LOB_PAGE_COUNT,
       SUM(lob_reserved_page_count)                                                          AS LOB_PAGE_USED,
       SUM(row_overflow_used_page_count)                                                     AS OVERFLOW_PAGE_USED
FROM   sys.tables t
       INNER JOIN sys.indexes i ON t.OBJECT_ID = i.object_id
       INNER JOIN sys.partitions p ON i.object_id = p.OBJECT_ID
           AND i.index_id = p.index_id
       INNER JOIN sys.allocation_units a ON p.partition_id = a.container_id
       INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
       INNER JOIN sys.dm_db_partition_stats d ON d.object_id = p.object_id
           AND d.index_id = p.index_id
           AND d.index_id = i.index_id
           AND d.partition_id = p.partition_id
           AND d.partition_number = p.partition_number
-- WHERE t.name in (N'GUIDANCE_1', N'GUIDANCE_2')
GROUP  BY t.NAME, s.NAME, p.Rows, i.name
ORDER  BY s.NAME, t.NAME, i.name
```

# Identify objects that use the most memory
```sql
SELECT COUNT(1)/128 AS megabytes_in_cache, name, index_id
FROM sys.dm_os_buffer_descriptors AS bd
INNER JOIN
(
  SELECT object_name(object_id) AS name, index_id, allocation_unit_id
  FROM sys.allocation_units AS au
  INNER JOIN sys.partitions AS p
  ON au.container_id = p.hobt_id
  AND (au.type = 1 OR au.type = 3)
  UNION ALL
  SELECT object_name(object_id) AS name, index_id, allocation_unit_id
  FROM sys.allocation_units AS au
  INNER JOIN sys.partitions AS p
  ON au.container_id = p.partition_id
  AND au.type = 2
) AS obj
ON bd.allocation_unit_id = obj.allocation_unit_id
WHERE database_id = DB_ID()
GROUP BY name, index_id
ORDER BY megabytes_in_cache DESC;
```

# File size
```sql
SELECT DB_NAME() AS DbName,
    name AS FileName,
    type_desc,
    size/128.0 AS CurrentSizeMB, 
    size/128.0 - CAST(FILEPROPERTY(name, 'SpaceUsed') AS INT)/128.0 AS FreeSpaceMB
FROM sys.database_files
WHERE type IN (0,1);
```

# Identify how many megabytes a database has in cache
```sql
SELECT
    CASE database_id
      WHEN 32767 THEN 'ResourceDb'
      ELSE db_name(database_id)
    END
  AS database_name, COUNT(1)/128 AS megabytes_in_cache
FROM sys.dm_os_buffer_descriptors
GROUP BY DB_NAME(database_id) ,database_id
ORDER BY megabytes_in_cache DESC;
```

# Provide information on missing indexes (very useful to execute on a database after running a slow-running job)
```sql
SELECT
  CONVERT (varchar(30), getdate(), 126) AS runtime,  mig.index_group_handle,  mid.index_handle,
  CONVERT (decimal (28, 1), migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) ) AS improvement_measure,
  'CREATE INDEX missing_index_' + CONVERT (varchar, mig.index_group_handle) + '_' + CONVERT (varchar, mid.index_handle) + ' ON ' + mid.statement + ' (' + ISNULL (mid.equality_columns, '') + CASE
    WHEN mid.equality_columns IS NOT NULL
    AND mid.inequality_columns IS NOT NULL THEN ','
    ELSE ''
  END + ISNULL (mid.inequality_columns, '') + ')' + ISNULL (' INCLUDE (' + mid.included_columns + ')', '') AS create_index_statement,
  migs.*, mid.database_id, mid.[object_id]
FROM sys.dm_db_missing_index_groups mig
    INNER JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
    INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE CONVERT (decimal (28, 1),migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans)) > 10
ORDER BY migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) DESC
```
