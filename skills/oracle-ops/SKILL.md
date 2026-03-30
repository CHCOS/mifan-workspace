---
name: oracle-ops
description: Oracle数据库巡检、运维与优化。当用户提到Oracle运维、Oracle巡检、数据库性能优化、Oracle RAC、表空间管理、AWR报告、SQL调优、Oracle备份恢复、RMAN、Data Guard、Oracle高可用、监听器管理、Oracle告警日志、Oracle死锁、Oracle参数调优时使用。
---

# Oracle 数据库运维

## 前置准备

```bash
# 环境变量
export ORACLE_SID=orcl
export ORACLE_HOME=/u01/app/oracle/product/19.3/dbhome_1
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH

# 验证连接
sqlplus -v
tnsping $ORACLE_SID
```

## 巡检 SQL（以 sysdba 执行）

### 1. 实例基础信息
```sql
-- 数据库版本和状态
SELECT * FROM v$version;
SELECT name, db_unique_name, open_mode, database_role, log_mode, created
FROM v$database;

SELECT instance_name, host_name, status, startup_time, version_full
FROM v$instance;

-- 字符集
SELECT * FROM nls_database_parameters WHERE parameter IN ('NLS_CHARACTERSET','NLS_NCHAR_CHARACTERSET');
```

### 2. 表空间使用情况
```sql
SELECT a.tablespace_name "表空间",
       ROUND(a.bytes/1024/1024/1024, 2) "总大小GB",
       ROUND(b.bytes/1024/1024/1024, 2) "已用GB",
       ROUND((a.bytes-b.bytes)/1024/1024/1024, 2) "使用GB",
       ROUND(b.bytes/a.bytes*100, 2) "使用率%"
FROM (SELECT tablespace_name, SUM(bytes) bytes FROM dba_data_files GROUP BY tablespace_name) a,
     (SELECT tablespace_name, SUM(bytes) bytes FROM dba_free_space GROUP BY tablespace_name) b
WHERE a.tablespace_name = b.tablespace_name
ORDER BY (a.bytes-b.bytes)/a.bytes DESC;

-- 临时表空间
SELECT tablespace_name, ROUND(used_space/1024/1024,2) "UsedMB",
       ROUND(total_space/1024/1024,2) "TotalMB",
       ROUND(used_space/total_space*100,2) "Used%"
FROM dba_temp_space_usage;

-- 自动扩展检查
SELECT tablespace_name, file_name, autoextensible, increment_by
FROM dba_data_files WHERE autoextensible = 'NO';

-- 数据文件路径
SELECT tablespace_name, file_name, ROUND(bytes/1024/1024/1024,2) size_gb, autoextensible
FROM dba_data_files ORDER BY tablespace_name;
```

### 3. ASM 使用情况（如使用ASM）
```sql
SELECT name, type, total_mb, free_mb, ROUND(free_mb/total_mb*100,2) free_pct
FROM v$asm_diskgroup;
```

### 4. 性能指标
```sql
-- 活动会话数
SELECT COUNT(*) FROM v$session WHERE status='ACTIVE';
SELECT sid, serial#, username, status, machine, program, sql_id
FROM v$session WHERE status='ACTIVE' ORDER BY last_call_et DESC;

-- 阻塞/锁等待
SELECT s.sid, s.serial#, s.username, s.status, s.machine,
       l.type, l.mode_held, l.mode_requested, l.block
FROM v$lock l JOIN v$session s ON l.sid = s.sid
WHERE l.block = 1;

-- 死锁
SELECT s1.sid, s1.serial#, s1.username "等待者",
       s2.sid, s2.serial#, s2.username "持有者",
       w.rowid, w.wait_event
FROM v$session_wait w, v$session s1, v$lock l1, v$lock l2, v$session s2
WHERE w.event = 'enqueue' AND w.sid = s1.sid
  AND l1.sid = s1.sid AND l2.id1 = l1.id1 AND l2.id2 = l1.id2
  AND l2.sid = s2.sid AND l2.block = 1;

-- 长事务
SELECT sid, serial#, status, username, machine,
       ROUND(t.used_ublk * 8192/1024/1024, 2) "UndoMB",
       TO_CHAR(start_date, 'YYYY-MM-DD HH24:MI:SS') start_time
FROM v$transaction t, v$session s, v$rollstat r
WHERE t.ses_addr = s.saddr AND t.xidusn = r.usn
ORDER BY t.used_ublk DESC;
```

### 5. 资源使用
```sql
-- SGA/PGA
SELECT name, ROUND(value/1024/1024,2) "MB"
FROM v$sga;
SELECT name, ROUND(value/1024/1024,2) "MB"
FROM v$pgastat WHERE name IN ('total PGA allocated','total PGA inuse','total freeable PGA memory');

-- 内存命中率
SELECT ROUND((1-SUM(getmisses)/SUM(gets))*100, 2) "LibraryCacheHit%"
FROM v$rowcache;
SELECT ROUND(SUM(pins-reloads)/SUM(pins)*100, 2) "LibraryPinHit%"
FROM v$librarycache;

-- Buffer Cache命中率
SELECT ROUND((1-phy.value/(cur.value+con.value))*100,2) "BufferHit%"
FROM v$sysstat cur, v$sysstat con, v$sysstat phy
WHERE cur.name='db block gets' AND con.name='consistent gets' AND phy.name='physical reads';
```

### 6. 告警日志
```bash
# 最近500行告警日志
tail -500 $ORACLE_HOME/diag/rdbms/$ORACLE_SID/$ORACLE_SID/trace/alert_$ORACLE_SID.log | grep -iE "ORA-|error|fatal|warn|corrupt"
# ORA错误统计
grep "ORA-" $ORACLE_HOME/diag/rdbms/$ORACLE_SID/$ORACLE_SID/trace/alert_$ORACLE_SID.log | awk -F'ORA-' '{print $2}' | awk '{print $1}' | sort | uniq -c | sort -rn | head -20
```

### 7. RMAN 备份状态
```sql
-- 最近备份
SELECT input_type, status, to_char(start_time,'YYYY-MM-DD HH24:MI') start_time,
       to_char(end_time,'YYYY-MM-DD HH24:MI') end_time,
       ROUND(elapsed_seconds/60,1) "Minutes",
       output_device_type, tag
FROM v$rman_status WHERE status='COMPLETED'
ORDER BY start_time DESC FETCH FIRST 10 ROWS ONLY;

-- 备份保留策略
SELECT recovery_window, redundancy FROM v$rman_configuration WHERE name LIKE '%retention%';

-- 归档日志空间
SELECT ROUND(SUM(space_used)/1024/1024/1024,2) "UsedGB",
       ROUND(SUM(space_limit)/1024/1024/1024,2) "LimitGB"
FROM v$recovery_file_dest;
```

### 8. Data Guard 状态（如配置DG）
```sql
SELECT name, db_unique_name, open_mode, database_role, protection_mode, protection_level, switchover_status
FROM v$database;
-- 延迟检查
SELECT name, value, datum_time, time_computed FROM v$dataguard_stats WHERE name LIKE 'apply lag%';
```

### 9. 无效对象和索引
```sql
-- 无效对象
SELECT owner, object_type, COUNT(*) invalid_count
FROM dba_objects WHERE status='INVALID' GROUP BY owner, object_type ORDER BY 2 DESC;
-- 重建
SELECT 'ALTER ' || object_type || ' ' || owner || '.' || object_name || ' COMPILE;' compile_sql
FROM dba_objects WHERE status='INVALID' AND object_type IN ('VIEW','PROCEDURE','FUNCTION','PACKAGE','TRIGGER');
```

## SQL 调优

### 执行计划分析
```sql
-- 获取执行计划
EXPLAIN PLAN FOR <sql>;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- 实际执行计划（已在cursor cache中）
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(sql_id=>'xxx', format=>'ALLSTATS LAST'));

-- AWR SQL报告
SELECT sql_id, executions, ROUND(elapsed_time/1000000,2) "ElapsedSec",
       ROUND(cpu_time/1000000,2) "CPUSec", disk_reads, buffer_gets,
       ROUND(elapsed_time/executions/1000000,2) "AvgSec",
       rows_processed, first_load_time
FROM v$sql WHERE sql_text LIKE '%关键字%' ORDER BY elapsed_time DESC;
```

### AWR 报告生成
```sql
-- 查看快照列表
SELECT snap_id, to_char(begin_interval_time,'YYYY-MM-DD HH24:MI') begin_time,
       to_char(end_interval_time,'YYYY-MM-DD HH24:MI') end_time
FROM dba_hist_snapshot ORDER BY snap_id DESC FETCH FIRST 10 ROWS ONLY;

-- 生成AWR报告
@?/rdbms/admin/awrrpt.sql
-- 生成AWR对比报告
@?/rdbms/admin/awrrpti.sql
-- 生成ASH报告
@?/rdbms/admin/ashrpt.sql
```

### ADDM 建议
```sql
SELECT task_name, status, to_char(advisor_run_time,'YYYY-MM-DD HH24:MI') run_time
FROM dba_advisor_tasks WHERE task_name LIKE 'ADDM%' ORDER BY advisor_run_time DESC;
-- 查看建议
SELECT dbms_advisor.get_task_report('ADDM:task_id') FROM dual;
```

## 常见运维操作

### 表空间扩容
```sql
-- 数据文件扩容
ALTER DATABASE DATAFILE '/u01/app/oracle/oradata/orcl/users01.dbf' RESIZE 10G;
-- 增加数据文件
ALTER TABLESPACE users ADD DATAFILE '/u01/app/oracle/oradata/orcl/users02.dbf' SIZE 1G AUTOEXTEND ON MAXSIZE 30G;
```

### 用户管理
```sql
-- 创建用户
CREATE USER app_user IDENTIFIED BY "StrongPassword123!" DEFAULT TABLESPACE users QUOTA 500M ON users;
GRANT CONNECT, RESOURCE TO app_user;

-- 密码过期检查
SELECT username, account_status, expiry_date, lock_date
FROM dba_users WHERE account_status != 'OPEN' ORDER BY expiry_date;

-- 解锁
ALTER USER app_user ACCOUNT UNLOCK;
```

### 统计信息收集
```sql
-- 整库统计
EXEC DBMS_STATS.GATHER_DATABASE_STATS(estimate_percent=>DBMS_STATS.AUTO_SAMPLE_SIZE, degree=>8);
-- 单表
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA','TABLE_NAME', cascade=>TRUE, degree=>4);
-- 检查过期统计
SELECT table_name, num_rows, last_analyzed FROM dba_tables
WHERE last_analyzed < SYSDATE - 7 AND num_rows > 10000 ORDER BY last_analyzed;
```

### 监听器管理
```bash
# 状态
lsnrctl status
# 启动/停止
lsnrctl start LISTENER_ORCL
lsnrctl stop
# 配置
cat $ORACLE_HOME/network/admin/listener.ora
cat $ORACLE_HOME/network/admin/tnsnames.ora
```

## 巡检报告模板

```markdown
# Oracle 数据库巡检报告

**实例:** orcl | **版本:** 19.3 | **角色:** Primary | **巡检时间:** 2026-03-30

## 1. 实例状态
| 指标 | 值 | 状态 |
|------|-----|------|
| Open Mode | READ WRITE | ✅ |
| Log Mode | ARCHIVELOG | ✅ |
| 运行时间 | xx天 | |

## 2. 资源使用
| 指标 | 值 | 阈值 | 状态 |
|------|-----|------|------|
| 表空间使用率(最高) | | < 85% | ✅/⚠️ |
| Buffer Cache命中率 | | > 95% | ✅/⚠️ |
| Library Cache命中率 | | > 95% | ✅/⚠️ |
| 活动会话数 | | | |
| 阻塞/锁等待 | | 0 | ✅/⚠️ |
| 无效对象数 | | 0 | ✅/⚠️ |

## 3. 备份状态
## 4. 告警日志摘要
## 5. Top SQL（按执行时间）
## 6. 异常与建议
```
