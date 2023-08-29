
# Some useful diagnostic commands
```
sp_who2 'active' - current activity on database
dbcc inputbuffer(SPID) - get detail information on SPID
```

# Capture active running SQLs on this database
```sql
-- Capture active running SQLs on this database
-- Can be used on primary database and read replica
-- This is a snapshot of the activities on the database on the database when SQL is executed.
SELECT s.session_id,
       r.STATUS,
       r.blocking_session_id                                        AS
       'blocked_by',
       r.wait_type,
       r.wait_resource,
       CONVERT(VARCHAR, Dateadd(ms, r.wait_time, 0), 8)             AS
       'wait_time',
       r.cpu_time,
       r.logical_reads,
       r.reads,
       r.writes,
       CONVERT(VARCHAR, (r.total_elapsed_time/1000 / 86400))
       + 'd '
       + CONVERT(VARCHAR, Dateadd(ms, r.total_elapsed_time, 0), 8)  AS
       'elapsed_time',
       Cast(( '<?query --  ' + Char(13) + Char(13)
              + Substring(st.TEXT, (r.statement_start_offset / 2) + 1, ( ( CASE
                   r.statement_end_offset WHEN - 1 THEN Datalength(st.TEXT) ELSE
                   r.statement_end_offset END -
                   r.statement_start_offset ) / 2 ) + 1)
              + Char(13) + Char(13) + '--?>' ) AS XML)              AS
       'query_text',
       COALESCE(Quotename(Db_name(st.dbid)) + N'.'
                + Quotename(OBJECT_SCHEMA_NAME(st.objectid, st.dbid))
                + N'.'
                + Quotename(Object_name(st.objectid, st.dbid)), '') AS
       'stored_proc'
       --,qp.query_plan AS 'xml_plan'  -- uncomment (1) if you want to see plan
       ,
       r.command,
       s.login_name,
       s.host_name,
       s.program_name,
       s.host_process_id,
       s.last_request_end_time,
       s.login_time,
       r.open_transaction_count
FROM   sys.dm_exec_sessions AS s
       INNER JOIN sys.dm_exec_requests AS r
               ON r.session_id = s.session_id
       CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS st
--OUTER APPLY sys.dm_exec_query_plan(r.plan_handle) AS qp -- uncomment (2) if you want to see plan
WHERE  r.wait_type NOT LIKE 'SP_SERVER_DIAGNOSTICS%'
        OR r.session_id != @@SPID
ORDER  BY r.cpu_time DESC,
          r.STATUS,
          r.blocking_session_id,
          s.session_id
```

# Top 10 longest-running SQLs including any blocks
```sql
-- The query below will display the top ten running queries that have the longest total elapsed time and are blocking other queries:
SELECT TOP 10
    r.session_id,r.plan_handle,r.sql_handle,r.request_id,r.start_time, r.status,r.command, r.database_id,r.user_id, r.wait_type
    ,r.wait_time,r.last_wait_type,r.wait_resource, r.total_elapsed_time,r.cpu_time, r.transaction_isolation_level,r.row_count,st.text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) as st 
WHERE r.blocking_session_id = 0 and r.session_id in (SELECT distinct(blocking_session_id) FROM sys.dm_exec_requests)
GROUP BY
    r.session_id, r.plan_handle,r.sql_handle, r.request_id,r.start_time, r.status,r.command, r.database_id,r.user_id, r.wait_type
    ,r.wait_time,r.last_wait_type,r.wait_resource, r.total_elapsed_time,r.cpu_time, r.transaction_isolation_level,r.row_count,st.text 
ORDER BY r.total_elapsed_time desc
```
