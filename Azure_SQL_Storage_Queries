

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
