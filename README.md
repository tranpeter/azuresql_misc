# azuresql_misc

This repo captures a bunch of useful Azure SQLs I use.

```
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
