---
name: postgresql-ops
description: PostgreSQL数据库巡检、运维与优化。当用户提到PostgreSQL运维、PG巡检、PostgreSQL性能优化、PostgreSQL主从复制、PostgreSQL备份恢复、pg_dump、WAL、VACUUM、流复制、逻辑复制、pg_stat_statements、pgBadger、PITR、PostgreSQL索引优化时使用。
---

# PostgreSQL 数据库运维

## 前置准备
```bash
sudo -u postgres psql -c "SELECT version();"
sudo -u postgres psql -c "SHOW port; SHOW data_directory;"
```

## 巡检 SQL

### 1. 实例基础
```sql
SELECT version();
SELECT pg_postmaster_start_time() uptime;
SELECT current_database(), current_user, inet_server_addr(), inet_server_port();
SHOW cluster_name;
SELECT name, setting FROM pg_settings WHERE name IN ('max_connections','shared_buffers','effective_cache_size','work_mem','maintenance_work_mem','wal_level','max_wal_size','checkpoint_completion_target');
```

### 2. 连接与锁
```sql
-- 连接数
SELECT count(*) FROM pg_stat_activity;
-- 空闲连接过多
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
-- 长事务
SELECT pid, now()-xact_start AS duration, query, state
FROM pg_stat_activity WHERE state IN ('idle in transaction','active')
AND now()-xact_start > interval '5 minutes';
-- 锁等待
SELECT blocked.pid, blocked.query, blocking.pid, blocking.query
FROM pg_locks bl JOIN pg_stat_activity blocked ON bl.pid=blocked.pid
JOIN pg_locks kl ON bl.locktype=kl.locktype AND bl.database IS NOT DISTINCT FROM kl.database
  AND bl.relation IS NOT DISTINCT FROM kl.relation AND bl.page IS NOT DISTINCT FROM kl.page
  AND bl.tuple IS NOT DISTINCT FROM kl.tuple AND bl.pid != kl.pid
JOIN pg_stat_activity blocking ON kl.pid=blocking.pid WHERE NOT bl.granted;
-- 死锁
SELECT * FROM pg_locks WHERE NOT granted;
```

### 3. 存储使用
```sql
-- 数据库大小
SELECT datname, pg_size_pretty(pg_database_size(datname)) size
FROM pg_database WHERE datistemplate=false ORDER BY pg_database_size(datname) DESC;

-- 表大小TOP20
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) total_size,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) data_size,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) idx_size
FROM pg_tables WHERE schemaname NOT IN ('pg_catalog','information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC LIMIT 20;

-- 表空间
SELECT spcname, pg_size_pretty(pg_tablespace_location(oid)) location FROM pg_tablespace;
-- 碎片
SELECT schemaname||'.'||tablename as table,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) bloat
FROM pg_tables WHERE schemaname NOT IN ('pg_catalog','information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename) DESC LIMIT 20;
```

### 4. 性能指标
```sql
-- 缓存命中率
SELECT sum(heap_blks_hit)/(sum(heap_blks_hit)+sum(heap_blks_read))*100 cache_hit_ratio FROM pg_statio_user_tables;
-- 索引使用率
SELECT sum(idx_scan) idx_scans, sum(idx_tup_fetch) idx_tup_fetch, sum(idx_tup_read) idx_tup_read FROM pg_stat_user_indexes;
-- 未使用索引
SELECT schemaname||'.'||relname AS table, indexrelname AS index, idx_scan
FROM pg_stat_user_indexes WHERE idx_scan = 0 ORDER BY pg_relation_size(indexrelid) DESC;
-- TOP SQL (按执行时间)
SELECT query, calls, total_time, mean_time, rows FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 20;
-- 顺序扫描过多
SELECT schemaname||'.'||relname, seq_scan, idx_scan, n_live_tup
FROM pg_stat_user_tables WHERE seq_scan > idx_scan AND n_live_tup > 1000 ORDER BY seq_scan DESC LIMIT 20;
```

### 5. WAL与检查点
```sql
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());
SELECT * FROM pg_stat_wal;
SELECT pg_walfile_name_offset(pg_current_wal_lsn());
SELECT count(*), pg_size_pretty(sum(size)) FROM pg_ls_waldir();
-- 检查点
SELECT * FROM pg_stat_bgwriter;
```

### 6. 流复制状态
```sql
-- 主库
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn,replay_lsn) lag_bytes
FROM pg_stat_replication;
-- 备库
SELECT status, sender_host, sender_port, conninfo FROM pg_stat_wal_receiver;
-- 复制延迟
SELECT client_addr, state,
       pg_wal_lsn_diff(sent_lsn,replay_lsn)/1024/1024 lag_mb,
       (EXTRACT(EPOCH FROM (now() - reply_time)))::numeric(10,2) lag_seconds
FROM pg_stat_replication;
```

### 7. VACUUM状态
```sql
SELECT relname, n_live_tup, n_dead_tup,
       ROUND(n_dead_tup::numeric/(CASE WHEN n_live_tup>0 THEN n_live_tup ELSE 1 END)*100,2) dead_ratio,
       last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;
```

### 8. 备份检查
```sql
-- pg_stat_archiver归档状态
SELECT * FROM pg_stat_archiver;
-- WAL归档延迟
SELECT archived_count, failed_count, last_failed_wal, last_archived_wal FROM pg_stat_archiver;
```

## 常见运维操作

### 备份恢复
```bash
# 逻辑备份
pg_dump -U postgres -F c -b -f full_backup_$(date +%Y%m%d).dump dbname
pg_dumpall -U postgres -f full_all_$(date +%Y%m%d).sql
# 恢复
pg_restore -U postgres -d dbname -c full_backup.dump

# 物理备份(pg_basebackup)
pg_basebackup -U replicator -h master_ip -D /backup/base -Fp -Xs -P -R

# PITR恢复
# recovery.target_time = '2026-03-30 10:00:00'
```

### VACUUM优化
```sql
-- 手动VACUUM
VACUUM ANALYZE verbose tablename;
-- 仅分析
ANALYZE tablename;
-- 回收空间(排他锁)
VACUUM FULL tablename;
```

### 参数优化建议
```sql
SELECT name, current_setting(name) val, source
FROM pg_settings
WHERE name IN ('shared_buffers','effective_cache_size','work_mem','maintenance_work_mem',
    'wal_buffers','checkpoint_completion_target','default_statistics_target','random_page_cost',
    'effective_io_concurrency','max_worker_processes','max_parallel_workers_per_gather',
    'max_parallel_workers','max_parallel_maintenance_workers','huge_pages','max_connections');
```

### 配置自动清理
```sql
-- 自动vacuum阈值
SHOW autovacuum_vacuum_threshold;
SHOW autovacuum_analyze_threshold;
SHOW autovacuum_vacuum_scale_factor;
-- 修改表级别
ALTER TABLE tablename SET (autovacuum_vacuum_scale_factor = 0.05);
ALTER TABLE tablename SET (autovacuum_analyze_scale_factor = 0.02);
```

## 巡检报告模板

```markdown
# PostgreSQL 巡检报告
**实例:** xxx | **版本:** 15.x | **端口:** 5432 | **运行时间:** xx天

## 资源使用
| 指标 | 值 | 阈值 | 状态 |
|------|-----|------|------|
| 数据库总大小 | | - | |
| 缓存命中率 | | > 99% | ✅/⚠️ |
| 活跃连接数 | | < max_connections×80% | ✅/⚠️ |
| 空闲连接数 | | - | |
| 长事务数 | | 0 | ✅/⚠️ |
| 锁等待数 | | 0 | ✅/⚠️ |
| 未使用索引数 | | - | |

## 流复制状态
## VACUUM/ANALYZE
## WAL/归档
## 异常与建议
```
