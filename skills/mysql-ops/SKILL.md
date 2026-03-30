---
name: mysql-ops
description: MySQL数据库巡检、运维与优化。当用户提到MySQL运维、MySQL巡检、MySQL性能优化、MySQL主从复制、MySQL备份恢复、MySQL配置调优、MySQL慢查询、MySQL索引优化、MySQL死锁、InnoDB引擎调优、mysqldump、binlog、GTID时使用。
---

# MySQL 数据库运维

## 前置准备
```bash
mysql -u root -p -e "SELECT VERSION();"
mysql -u root -p -e "SHOW VARIABLES LIKE 'port';"
```

## 巡检 SQL

### 1. 实例基础
```sql
SELECT VERSION(), CURRENT_USER(), @@hostname, @@port, @@datadir;
SHOW GLOBAL VARIABLES LIKE 'innodb_version';
SHOW GLOBAL STATUS LIKE 'Uptime';
```

### 2. 连接与线程
```sql
SHOW GLOBAL STATUS LIKE 'Threads_%';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
SHOW GLOBAL VARIABLES LIKE 'max_connections';
SHOW PROCESSLIST;
-- 长事务
SELECT * FROM information_schema.innodb_trx ORDER BY trx_started;
-- 锁等待
SELECT * FROM sys.schema_table_lock_waits;
SELECT r.trx_id waiting_trx, r.trx_mysql_thread_id waiting_thread,
       b.trx_id blocking_trx, b.trx_mysql_thread_id blocking_thread
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;
-- 死锁
SHOW ENGINE INNODB STATUS;
SELECT * FROM sys.schema_table_statistics_with_buffer ORDER BY rows_sent DESC LIMIT 10;
```

### 3. 存储使用
```sql
-- 表空间
SELECT table_schema, ROUND(SUM(data_length+index_length)/1024/1024/1024,2) "SizeGB",
       ROUND(SUM(data_length)/1024/1024/1024,2) "DataGB",
       ROUND(SUM(index_length)/1024/1024/1024,2) "IndexGB",
       COUNT(*) table_count
FROM information_schema.tables WHERE table_schema NOT IN ('mysql','sys','information_schema','performance_schema')
GROUP BY table_schema ORDER BY 2 DESC;

-- 大表TOP10
SELECT table_schema, table_name, table_rows,
       ROUND(data_length/1024/1024,2) data_mb, ROUND(index_length/1024/1024,2) index_mb
FROM information_schema.tables WHERE table_schema NOT IN ('mysql','sys','information_schema','performance_schema')
ORDER BY data_length+index_length DESC LIMIT 10;

-- 碎片空间
SELECT table_schema, table_name, data_free/1024/1024 free_mb
FROM information_schema.tables WHERE data_free > 1024*1024 ORDER BY data_free DESC LIMIT 20;
```

### 4. 性能指标
```sql
-- InnoDB缓冲池
SELECT @@innodb_buffer_pool_size/1024/1024/1024 pool_gb;
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';
-- 缓冲池命中率
SELECT ROUND((1 - SUM(VARIABLE_VALUE)/SUM(NULLIF(VARIABLE_VALUE,0)))*100, 2) 'BufferPoolHit%'
FROM (SELECT VARIABLE_VALUE FROM sys.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests') r,
     (SELECT VARIABLE_VALUE FROM sys.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_reads') rd;

-- 慢查询
SHOW GLOBAL STATUS LIKE 'Slow_queries';
SHOW GLOBAL VARIABLES LIKE 'slow_query_log%';
SHOW GLOBAL VARIABLES LIKE 'long_query_time';
SELECT * FROM sys.statements_with_runtimes_in_95th_percentile LIMIT 10;

-- QPS/TPS
SHOW GLOBAL STATUS LIKE 'Com_%';
SELECT SUM(IF(variable_name='Com_select',variable_value,0)) selects,
       SUM(IF(variable_name='Com_insert',variable_value,0)) inserts,
       SUM(IF(variable_name='Com_update',variable_value,0)) updates,
       SUM(IF(variable_name='Com_delete',variable_value,0)) deletes
FROM performance_schema.global_status WHERE variable_name LIKE 'Com_%';

-- 临时表
SHOW GLOBAL STATUS LIKE 'Created_tmp%';

-- 全表扫描
SELECT * FROM sys.statements_with_full_table_scans ORDER BY rows_examined DESC LIMIT 10;
```

### 5. 主从复制
```sql
SHOW SLAVE STATUS\G
SHOW MASTER STATUS\G
SHOW GLOBAL VARIABLES LIKE 'gtid%';
SHOW GLOBAL VARIABLES LIKE 'server_%';
-- 复制延迟
SHOW GLOBAL STATUS LIKE 'Seconds_Behind_Master';
-- GTID不一致检查
SELECT @@GLOBAL.GTID_EXECUTED;
SELECT @@GLOBAL.GTID_PURGED;
```

### 6. 备份检查
```sql
SHOW GLOBAL VARIABLES LIKE 'log_bin%';
SHOW GLOBAL STATUS LIKE 'Binlog%';
SELECT COUNT(*) binlog_count,
       ROUND(SUM(file_size)/1024/1024/1024,2) total_gb
FROM information_schema.INNODB_SYS_TABLESPACES WHERE name LIKE 'mysql/innodb_%';
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
```

## 常见运维操作

### 参数优化建议
```sql
-- 关键参数检查
SELECT variable_name, variable_value
FROM information_schema.global_variables
WHERE variable_name IN ('innodb_buffer_pool_size','innodb_log_file_size','innodb_flush_log_at_trx_commit',
    'innodb_flush_method','sync_binlog','max_connections','thread_cache_size','table_open_cache',
    'tmp_table_size','max_heap_table_size','sort_buffer_size','join_buffer_size','read_rnd_buffer_size',
    'innodb_io_capacity','innodb_io_capacity_max','innodb_read_io_threads','innodb_write_io_threads',
    'query_cache_size','query_cache_type','log_slow_queries','long_query_time');
```

### 慢查询分析
```bash
mysqldumpslow -s t /var/log/mysql/slow.log | head -20
pt-query-digest /var/log/mysql/slow.log --limit 10
```

### 索引优化
```sql
-- 未使用索引
SELECT * FROM sys.schema_unused_indexes;
-- 冗余索引
SELECT * FROM sys.schema_redundant_indexes;
-- 缺失索引建议
SELECT * FROM sys.schema_index_statistics WHERE rows_selected > rows_read;
```

### 备份恢复
```bash
# 逻辑备份
mysqldump -u root -p --single-transaction --routines --triggers --events --all-databases > full_backup_$(date +%Y%m%d).sql
# 单库
mysqldump -u root -p --single-transaction dbname > dbname_$(date +%Y%m%d).sql

# 使用xtrabackup
xtrabackup --backup --target-dir=/backup/full --user=root --password=xxx
xtrabackup --prepare --target-dir=/backup/full
xtrabackup --copy-back --target-dir=/backup/full
```

## 巡检报告模板

```markdown
# MySQL 数据库巡检报告
**实例:** xxx | **版本:** 8.0.xx | **端口:** 3306 | **运行时间:** xx天

## 资源使用
| 指标 | 值 | 阈值 | 状态 |
|------|-----|------|------|
| QPS | | - | |
| TPS | | - | |
| 连接数/最大连接 | | < 80% | ✅/⚠️ |
| Buffer Pool命中率 | | > 95% | ✅/⚠️ |
| 慢查询数 | | 0 | ✅/⚠️ |
| 活跃线程数 | | - | |
| 锁等待 | | 0 | ✅/⚠️ |

## 主从状态
## 磁盘使用
## 异常与建议
```
