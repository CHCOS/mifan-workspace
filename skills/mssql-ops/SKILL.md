---
name: mssql-ops
description: SQL Server数据库巡检、运维与优化。当用户提到SQL Server运维、MSSQL巡检、SQL Server性能优化、SQL Server备份恢复、SQL Server AlwaysOn、SSMS、SQL Server代理作业、数据库镜像、复制发布订阅、SQL Server索引重建、死锁分析、数据库收缩、SQL Server安全时使用。
---

# SQL Server 数据库运维

## 巡检 SQL（SSMS / sqlcmd 执行）

### 1. 实例基础
```sql
SELECT @@VERSION;
SELECT SERVERPROPERTY('ServerName') ServerName, SERVERPROPERTY('Edition') Edition,
       SERVERPROPERTY('ProductLevel') ProductLevel, SERVERPROPERTY('ProductVersion') Version;
SELECT @@SPID, DB_NAME(), SYSTEM_USER, GETDATE() CurrentTime;
-- 实例启动时间
SELECT sqlserver_start_time FROM sys.dm_os_sys_info;
-- CPU核心数
SELECT cpu_count FROM sys.dm_os_sys_info;
```

### 2. 数据库状态
```sql
SELECT name, state_desc, recovery_model_desc, log_reuse_wait_desc,
       compatibility_level, page_verify_option_desc,
       ROUND(size*8/1024,2) SizeMB, ROUND(space*8/1024,2) SpaceMB,
       user_access_desc, is_read_only, is_auto_close_on, is_auto_shrink_on
FROM sys.master_files f
JOIN sys.databases d ON f.database_id = d.database_id
WHERE f.type = 0 ORDER BY f.size DESC;

-- 数据库文件详情
SELECT DB_NAME(database_id) DBName, name logical_name, type_desc, physical_name,
       ROUND(size*8/1024,2) SizeMB, ROUND(max_size*8/1024,2) MaxMB, growth, is_percent_growth
FROM sys.master_files ORDER BY database_id, type;
```

### 3. 连接与会话
```sql
-- 当前连接数
SELECT COUNT(*) FROM sys.dm_exec_connections;
-- 按数据库统计
SELECT DB_NAME(database_id) DBName, COUNT(*) Sessions
FROM sys.dm_exec_connections GROUP BY database_id ORDER BY 2 DESC;
-- 阻塞
SELECT blocking_session_id, session_id, wait_type, wait_time, wait_resource,
       blocking_these = STUFF((SELECT ', '+CAST(session_id AS VARCHAR) FROM sys.dm_exec_sessions
       WHERE blocking_session_id = s.session_id FOR XML PATH('')),1,1,'')
FROM sys.dm_exec_sessions s WHERE blocking_session_id != 0;
-- 死锁
SELECT * FROM sys.dm_xe_sessions WHERE name = 'system_health';
-- 长事务
SELECT session_id, transaction_id, transaction_begin_time,
       DATEDIFF(MINUTE, transaction_begin_time, GETDATE()) minutes_running, database_id
FROM sys.dm_tran_active_transactions WHERE transaction_begin_time < DATEADD(MINUTE,-30,GETDATE());
```

### 4. 磁盘与文件
```sql
-- 磁盘空间
SELECT DISTINCT SUBSTRING(physical_name,1,3) Drive,
       CAST(SUBSTRING(physical_name,1,1)+':\' AS VARCHAR) Drive2
FROM sys.master_files;
-- 驱动器可用空间
EXEC master..xp_fixeddrives;
-- 文件组空间
SELECT DB_NAME(database_id) DBName, name FileGroup, type_desc,
       ROUND(size*8/1024,2) UsedMB
FROM sys.database_files;
```

### 5. 性能指标
```sql
-- Buffer Pool命中率
SELECT ROUND((CAST(a.cntr_value AS FLOAT)/b.cntr_value)*100,2) BufferCacheHitRatio
FROM (SELECT cntr_value FROM sys.dm_os_performance_counters WHERE counter_name='Buffer cache hit ratio' AND object_name LIKE '%Buffer Manager%') a,
     (SELECT cntr_value FROM sys.dm_os_performance_counters WHERE counter_name='Buffer cache hit ratio base' AND object_name LIKE '%Buffer Manager%') b;

-- Page Life Expectancy
SELECT cntr_value FROM sys.dm_os_performance_counters WHERE counter_name='Page life expectancy';

-- 高CPU查询
SELECT TOP 10 total_worker_time/1000 cpu_ms, execution_count,
       SUBSTRING(t.text,1,200) query_text, db_name(t.dbid) db
FROM sys.dm_exec_query_stats qs CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
ORDER BY total_worker_time DESC;

-- 高IO查询
SELECT TOP 10 total_logical_reads, total_logical_writes, execution_count,
       SUBSTRING(t.text,1,200) query_text
FROM sys.dm_exec_query_stats qs CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
ORDER BY total_logical_reads DESC;

-- 等待统计
SELECT TOP 10 wait_type, waiting_tasks_count, wait_time_ms,
       ROUND(wait_time_ms/1000.0,2) wait_sec, signal_wait_time_ms
FROM sys.dm_os_wait_stats WHERE wait_type NOT IN ('SQLTRACE_BUFFER_FLUSH','LAZYWRITER_SLEEP')
ORDER BY wait_time_ms DESC;

-- 索引碎片
SELECT OBJECT_NAME(i.object_id, DB_ID()) TableName, i.name IndexName,
       index_type_desc, avg_fragmentation_in_percent, page_count
FROM sys.dm_db_index_physical_stats(DB_ID(),NULL,NULL,NULL,'LIMITED') ps
JOIN sys.indexes i ON ps.object_id=i.object_id AND ps.index_id=i.index_id
WHERE avg_fragmentation_in_percent > 30 AND page_count > 1000
ORDER BY avg_fragmentation_in_percent DESC;
```

### 6. AlwaysOn 状态
```sql
SELECT ag.name AGName, ags.primary_replica, ar.replica_server_name,
       ar.replica_id, ar.availability_mode_desc, ar.failover_mode_desc,
       ar.synchronization_health_desc, ar.connected_state_desc
FROM sys.availability_groups ag JOIN sys.availability_replicas ar ON ag.group_id=ar.group_id
LEFT JOIN sys.dm_hadr_availability_group_states ags ON ag.group_id=ags.group_id;

-- 同步延迟
SELECT drc.replica_id, drs.database_id, drs.synchronization_state_desc,
       drs.synchronization_health_desc, drs.database_state_desc,
       drs.log_send_queue_size, drs.redo_queue_size
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.dm_hadr_database_replica_cluster_states drc ON drs.replica_id=drc.replica_id AND drs.group_database_id=drc.group_database_id;
```

### 7. 作业状态
```sql
SELECT j.name JobName, j.enabled, ja.run_requested_date LastRun, ja.run_duration,
       CASE ja.run_status WHEN 0 THEN 'Failed' WHEN 1 THEN 'Succeeded' WHEN 3 THEN 'Cancelled' WHEN 4 THEN 'Retry' END Status
FROM msdb.dbo.sysjobs j LEFT JOIN msdb.dbo.sysjobactivity ja ON j.job_id=ja.job_id
WHERE ja.session_id=(SELECT MAX(session_id) FROM msdb.dbo.sysjobactivity)
ORDER BY j.name;
```

### 8. 备份状态
```sql
SELECT d.name DBName, MAX(b.backup_finish_date) LastBackup,
       DATEDIFF(HOUR, MAX(b.backup_finish_date), GETDATE()) HoursSinceBackup
FROM sys.databases d LEFT JOIN msdb.dbo.backupset b ON d.name=b.database_name
WHERE d.database_id > 4 AND d.state = 0
GROUP BY d.name ORDER BY 3 DESC;
```

### 9. 错误日志
```sql
EXEC xp_readerrorlog 0, 1, N'Error', NULL, DATEADD(DAY,-1,GETDATE()), NULL;
EXEC xp_readerrorlog 0, 1, N'Fail', NULL, DATEADD(DAY,-1,GETDATE()), NULL;
```

## 常见运维操作

### 索引维护
```sql
-- 重建碎片>30%
ALTER INDEX index_name ON schema.table REBUILD WITH (ONLINE = ON);
-- 重组碎片10-30%
ALTER INDEX index_name ON schema.table REORGANIZE;
-- 批量重建
DECLARE @sql NVARCHAR(MAX)='';
SELECT @sql+='ALTER INDEX '+i.name+' ON '+s.name+'.'+o.name+' REBUILD;' + CHAR(13)
FROM sys.dm_db_index_physical_stats(DB_ID(),NULL,NULL,NULL,'LIMITED') ps
JOIN sys.indexes i ON ps.object_id=i.object_id AND ps.index_id=i.index_id
JOIN sys.objects o ON i.object_id=o.object_id
JOIN sys.schemas s ON o.schema_id=s.schema_id
WHERE ps.avg_fragmentation_in_percent>30 AND ps.page_count>1000;
EXEC sp_executesql @sql;
```

### 数据库收缩
```sql
DBCC SHRINKDATABASE(dbname, 10);
DBCC SHRINKFILE(logical_name, target_size_mb);
```

### 维护计划
```sql
-- 数据库完整性检查
DBCC CHECKDB(dbname) WITH NO_INFOMSGS, ALL_ERRORMSGS;
-- 更新统计
UPDATE STATISTICS schema.table WITH FULLSCAN;
-- 重建索引
ALTER INDEX ALL ON schema.table REBUILD;
```

## 巡检报告模板

```markdown
# SQL Server 巡检报告
**实例:** xxx | **版本:** 2019 Enterprise | **运行时间:** xx天

## 资源使用
| 指标 | 值 | 阈值 | 状态 |
|------|-----|------|------|
| Buffer Cache命中率 | | > 95% | ✅/⚠️ |
| Page Life Expectancy | | > 300 | ✅/⚠️ |
| 活跃连接数 | | - | |
| 阻塞会话 | | 0 | ✅/⚠️ |
| 碎片>30%的索引 | | 0 | ✅/⚠️ |

## 数据库状态
## AlwaysOn状态
## 作业状态
## 备份状态
## 异常与建议
```
