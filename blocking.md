# Identify blocking SQL
```sql
SELECT
   [HeadBlocker] =
        CASE
            -- session has an active request, is blocked, but is blocking others
            -- or session is idle but has an open transaction and is blocking others
            WHEN r2.session_id IS NOT NULL AND (r.blocking_session_id = 0 OR r.session_id IS NULL) THEN '1'
            -- session is either not blocking someone, or is blocking someone but is blocked by another party
            ELSE ''
        END,
   [SessionID] = s.session_id,
   [Login] = s.login_name,  
   [Database] = db_name(p.dbid),
   [BlockedBy] = w.blocking_session_id,
   [OpenTransactions] = r.open_transaction_count,
   [Status] = s.status,
   [WaitType] = w.wait_type,
   [WaitTime_ms] = w.wait_duration_ms,
   [WaitResource] = r.wait_resource,
   [WaitResourceDesc] = w.resource_description,
   [Command] = r.command,
   [Application] = s.program_name,
   [TotalCPU_ms] = s.cpu_time,
   [TotalPhysicalIO_MB] = (s.reads + s.writes) * 8 / 1024,
   [MemoryUse_KB] = s.memory_usage * 8192 / 1024,
   [LoginTime] = s.login_time,
   [LastRequestStartTime] = s.last_request_start_time,
   [HostName] = s.host_name,
   [QueryHash] = r.query_hash,
   [BlockerQuery_or_MostRecentQuery] = txt.text
FROM sys.dm_exec_sessions s
LEFT OUTER JOIN sys.dm_exec_connections c ON (s.session_id = c.session_id)
LEFT OUTER JOIN sys.dm_exec_requests r ON (s.session_id = r.session_id)
LEFT OUTER JOIN sys.dm_os_tasks t ON (r.session_id = t.session_id AND r.request_id = t.request_id)
LEFT OUTER JOIN
(
    SELECT *, ROW_NUMBER() OVER (PARTITION BY waiting_task_address ORDER BY wait_duration_ms DESC) AS row_num
    FROM sys.dm_os_waiting_tasks
) w ON (t.task_address = w.waiting_task_address) AND w.row_num = 1
LEFT OUTER JOIN sys.dm_exec_requests r2 ON (s.session_id = r2.blocking_session_id)
LEFT OUTER JOIN sys.sysprocesses p ON (s.session_id = p.spid)
OUTER APPLY sys.dm_exec_sql_text (ISNULL(r.[sql_handle], c.most_recent_sql_handle)) AS txt
WHERE s.is_user_process = 1
AND (r2.session_id IS NOT NULL AND (r.blocking_session_id = 0 OR r.session_id IS NULL))
OR blocked > 0
ORDER BY [HeadBlocker] desc, s.session_id;
 
 
SELECT
   [s_tst].[session_id],
   [database_name] = DB_NAME (s_tdt.database_id),
   [s_tdt].[database_transaction_begin_time],
   [sql_text] = [s_est].[text]
FROM sys.dm_tran_database_transactions [s_tdt]
INNER JOIN sys.dm_tran_session_transactions [s_tst] ON [s_tst].[transaction_id] = [s_tdt].[transaction_id]
INNER JOIN sys.dm_exec_connections [s_ec] ON [s_ec].[session_id] = [s_tst].[session_id]
CROSS APPLY sys.dm_exec_sql_text ([s_ec].[most_recent_sql_handle]) AS [s_est];
 
SELECT tst.session_id, [database_name] = db_name(s.database_id)
, tat.transaction_begin_time
, transaction_duration_s = datediff(s, tat.transaction_begin_time, sysdatetime())
, input_buffer = ib.event_info, tat.transaction_uow    
, transaction_type = CASE tat.transaction_type  WHEN 1 THEN 'Read/write transaction'
                                                WHEN 2 THEN 'Read-only transaction'
                                                WHEN 3 THEN 'System transaction'
                                                WHEN 4 THEN 'Distributed transaction' END
, transaction_state  = CASE tat.transaction_state   
            WHEN 0 THEN 'The transaction has not been completely initialized yet.'
            WHEN 1 THEN 'The transaction has been initialized but has not started.'
            WHEN 2 THEN 'The transaction is active - has not been committed or rolled back.'
            WHEN 3 THEN 'The transaction has ended. This is used for read-only transactions.'
            WHEN 4 THEN 'The commit process has been initiated on the distributed transaction.'
            WHEN 5 THEN 'The transaction is in a prepared state and waiting resolution.'
            WHEN 6 THEN 'The transaction has been committed.'
            WHEN 7 THEN 'The transaction is being rolled back.'
            WHEN 8 THEN 'The transaction has been rolled back.' END
, transaction_name = tat.name, request_status = r.status
, azure_dtc_state = CASE tat.dtc_state
                    WHEN 1 THEN 'ACTIVE'
                    WHEN 2 THEN 'PREPARED'
                    WHEN 3 THEN 'COMMITTED'
                    WHEN 4 THEN 'ABORTED'
                    WHEN 5 THEN 'RECOVERED' END
, tst.is_user_transaction, tst.is_local
, session_open_transaction_count = tst.open_transaction_count 
, s.host_name, s.program_name, s.client_interface_name, s.login_name, s.is_user_process
FROM sys.dm_tran_active_transactions tat
INNER JOIN sys.dm_tran_session_transactions tst  on tat.transaction_id = tst.transaction_id
INNER JOIN sys.dm_exec_sessions s on s.session_id = tst.session_id
LEFT OUTER JOIN sys.dm_exec_requests r on r.session_id = s.session_id
CROSS APPLY sys.dm_exec_input_buffer(s.session_id, null) AS ib;
 
SELECT
   table_name = schema_name(o.schema_id) + '.' + o.name,
   wt.wait_duration_ms, wt.wait_type, wt.blocking_session_id, wt.resource_description,
   tm.resource_type, tm.request_status, tm.request_mode, tm.request_session_id
FROM sys.dm_tran_locks AS tm
INNER JOIN sys.dm_os_waiting_tasks as wt ON tm.lock_owner_address = wt.resource_address
LEFT OUTER JOIN sys.partitions AS p on p.hobt_id = tm.resource_associated_entity_id
LEFT OUTER JOIN sys.objects o on o.object_id = p.object_id or tm.resource_associated_entity_id = o.object_id
WHERE resource_database_id = DB_ID()
;
```
