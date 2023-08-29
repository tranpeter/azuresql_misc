# azuresql_misc

This repo captures a bunch of useful Azure SQLs I use.

```sql
-- Useful to find out whether the hostname and utilization. This query is used to find the database and instance CPU utilization.
-- This query is handy to monitor the read-replica since this is not readily exposed on Azure Portal.
-- Note the following 3 SQLs are usually executed together the first time to verify host_name and whether the database is read-only.
-- Note use: ApplicationIntent=READONLY  
 
DECLARE
    @hostname nvarchar(128),
    @updateability sql_variant;
BEGIN
 
    -- To verify you are connected to the read only replica:
    SELECT @updateability = DATABASEPROPERTYEX(DB_NAME(), 'Updateability');
     
    -- To get the hostname the following query works as a good way
    -- (It lists the hostname for the client application for all sessions but can help tell the replica you are connected to. eg DB42 etc.)
    SELECT DISTINCT @hostname = host_name from sys.dm_exec_sessions
    WHERE host_name not like 'price-guidance%' and host_name like 'DB%';
     
    -- Get the database and instance CPU utilization by Central Time zone
    select @hostname as hostname, @updateability as updateability, DB_NAME(),
          DATEADD(HH, -5, end_time) end_time_cst_tz, avg_cpu_percent, avg_instance_cpu_percent
    from sys.dm_db_resource_stats
    order by end_time desc;
END;
```


```sql
-- When this is used for fixing suddenly poor response times from guidance,
-- RUN this on the READ-REPLICAS
 
-- This can be executed against read-only replicas. Each replica will parse and generate the execution plan
-- independently of any other replica. It's possible to have different execution plans (highly unlikely)
-- for the same query across different replicas.
ALTER DATABASE SCOPED CONFIGURATION CLEAR PROCEDURE_CACHE;
```

```sql
-- Get the number of connections to the database.
--
select a.dbid,a.program_name, count(a.dbid) as TotalConnections
from sys.sysprocesses a
inner join sys.databases b on a.dbid = b.database_id
group by a.dbid, a.program_name;
```

```sql
--
-- Get SQL text by partial matching
--
SELECT Txt.query_text_id, Txt.query_sql_text, Pl.plan_id, Pl.query_plan, Qry.*
FROM sys.query_store_plan AS Pl
INNER JOIN sys.query_store_query AS Qry
    ON Pl.query_id = Qry.query_id
INNER JOIN sys.query_store_query_text AS Txt
    ON Qry.query_text_id = Txt.query_text_id
WHERE Txt.query_sql_text LIKE '(@P1 NVARCHAR(4000),@P2 INT,@P0 INT)SELECT TOP(@P0) CUSTOMER_ID AS CUST_ID%'
 
-- OR

--
-- Get SQL text by query id (when Query Store isn't available)
--
SELECT Txt.query_text_id, Txt.query_sql_text, Pl.plan_id, Pl.query_plan, Qry.*
FROM sys.query_store_plan AS Pl
INNER JOIN sys.query_store_query AS Qry
    ON Pl.query_id = Qry.query_id
INNER JOIN sys.query_store_query_text AS Txt
    ON Qry.query_text_id = Txt.query_text_id
WHERE Qry.query_id = <ID>
```

```sql
--
-- The following SQL will list out the databases in the given elastic pool sorted by most active CPU utilization.
-- NOTE, this query should either be executed on a database either in the primary (read/write) or replica (read-only).
-- If the query is executed on a replica database, the result will be based on all the other replica databases.
-- This means the query result WILL BE DIFFERENT if executed on a primary (read/write) vs. replica (read-only) database.
--
select group_id,
       name,
       g.database_name,
       Min(snapshot_time) as MinTime,
       Max(snapshot_time) as MaxTime,
       Sum(delta_cpu_active_ms) / Sum(duration_ms / 1000.0) as AvgDeltaCPU_Active_ms_persec,
       Sum(delta_cpu_usage_ms) / Sum(duration_ms / 1000.0) as AvgDeltaCPU_Usage_ms_persec,
       Sum(delta_background_writes) / Sum(duration_ms / 1000.0) as avgWrites_PerSec,
       Sum(delta_reads_issued) / Sum(duration_ms / 1000.0) as AvgReads_perSec,
       Sum(delta_log_bytes_used / 1024.0 / 1024.0) / Sum(duration_ms / 1000.0) as avg_log_mb_persec,
       Avg(max_io) as MaxIO_avg,Avg(max_log_rate_kb / 1024.0) as max_log_rate_mb
from   sys.dm_resource_governor_workload_groups_history_ex e,
       sys.dm_user_db_resource_governance g
where  name like 'UserPrimary%'
       and name = 'UserPrimaryGroup.DBId' + convert(varchar(10), g.database_id)
group  by name,group_id,g.database_name
order  by AvgDeltaCPU_Active_ms_persec desc;
```
