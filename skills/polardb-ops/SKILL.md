---
name: polardb-ops
description: PolarDB数据库巡检、运维与优化。当用户提到PolarDB运维、PolarDB巡检、PolarDB MySQL版、PolarDB PostgreSQL版、PolarDB性能优化、PolarDB备份恢复、PolarDB只读节点、PolarDB集群管理、PolarDB存储计费、PolarDB参数调优、PolarDB慢SQL、阿里云RDS/PolarDB管控台操作时使用。
---

# PolarDB 数据库运维

PolarDB 兼容 MySQL 和 PostgreSQL 两种引擎，巡检时先确认引擎版本。

## 前置确认
```sql
SELECT VERSION();
-- MySQL版显示: 5.7.36-log / 8.0.18 / 8.4.x
-- PG版显示: PostgreSQL 14.x / 15.x
SHOW engine;  -- MySQL版确认存储引擎
```

## 一、PolarDB MySQL版巡检

### 集群状态（阿里云管控台 + SQL）
```sql
-- 只读节点延迟
SHOW SLAVE HOSTS;
SELECT * FROM information_schema.replica_host_status;

-- 节点角色
SELECT server_id, server_type, role FROM mysql.server_nodes;

-- 连接池状态
SHOW GLOBAL STATUS LIKE 'PolarDB_connection_pool%';
```

### 存储与计算分离指标
```sql
-- 共享存储使用（通过管控台查看，或）
SELECT ROUND(SUM(data_length+index_length)/1024/1024/1024,2) total_gb
FROM information_schema.tables WHERE table_schema NOT IN ('mysql','sys','information_schema','performance_schema');

-- 一致性级别
SELECT @@group_replication_consistency;

-- 闪回查询
SELECT * FROM information_schema.innodb_trx ORDER BY trx_id DESC LIMIT 5;
```

### PolarDB特有性能
```sql
-- 并行查询
SHOW VARIABLES LIKE 'px%';
SHOW GLOBAL STATUS LIKE 'PolarDB_px%';
-- 全局一致性读
SHOW VARIABLES LIKE 'transaction_read_only';
SHOW VARIABLES LIKE 'tx_read_only';

-- 慢SQL（PolarDB内置SQL审计）
SELECT * FROM information_schema.mysql_slow_log
WHERE start_time > DATE_SUB(NOW(), INTERVAL 24 HOUR)
ORDER BY query_time DESC LIMIT 20;

-- 并行查询资源使用
SHOW GLOBAL STATUS LIKE 'PolarDB_parallel%';
```

## 二、PolarDB PostgreSQL版巡检

### 集群与存储
```sql
SELECT * FROM polar_cluster_info;  -- 集群信息
SELECT * FROM polar_node_info;     -- 节点信息

-- 只读节点延迟
SELECT application_name, client_addr, state, sent_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) lag_bytes
FROM pg_stat_replication;
```

## 三、通用运维操作

### 参数调优
```sql
-- MySQL版PolarDB特有参数
SHOW VARIABLES LIKE 'loose_polar_%';
-- 关键参数
SELECT variable_name, variable_value
FROM information_schema.global_variables
WHERE variable_name LIKE '%polar%' OR variable_name LIKE '%px%'
ORDER BY variable_name;

-- PG版PolarDB参数
SELECT name, setting FROM pg_settings WHERE name LIKE '%polar%' OR name LIKE '%px%';
```

### 只读节点管理
- 通过阿里云管控台 → PolarDB集群 → 只读节点
- 查看延迟：`SHOW SLAVE HOSTS` 或 `pg_stat_replication`
- 一致性级别：Session级别 / Global级别 / Eventual

### 备份恢复
- PolarDB基于快照备份，通过阿里云管控台操作
- 按时间点恢复：管控台 → 克隆/恢复 → 按时间点
- 跨地域备份：管控台 → 备份恢复 → 跨地域备份

### 存储空间管理
- PolarDB按存储空间计费（自动扩容，无需手动管理表空间）
- 监控存储使用：管控台 → 监控与报警 → 存储空间
- 手动扩容存储：管控台 → 变更配置 → 存储空间

## 四、监控告警

### 关键监控指标（阿里云CloudMonitor）
| 指标 | 说明 | 建议阈值 |
|------|------|----------|
| CPU使用率 | 主节点/只读节点 | < 80% |
| 内存使用率 | 主节点/只读节点 | < 85% |
| IOPS | 磁盘IO | 根据规格 |
| 存储空间 | 共享存储 | < 80% |
| 只读延迟 | 只读节点与主节点差 | < 1秒 |
| 活跃连接数 | 数据库连接数 | < max_connections×80% |
| 慢SQL数 | 慢查询数量 | 0 |
| TPS/QPS | 吞吐量 | 根据业务 |

## 五、常见问题排查

### 只读节点延迟大
```sql
-- MySQL版：检查并行回放
SHOW GLOBAL VARIABLES LIKE 'slave_parallel_workers';
SHOW GLOBAL STATUS LIKE 'Slave_seconds%';

-- PG版：检查WAL回放速度
SELECT pg_wal_lsn_diff(sent_lsn,replay_lsn)/1024/1024 lag_mb FROM pg_stat_replication;
```

### 性能下降
```sql
-- 检查并行查询是否生效
SHOW GLOBAL STATUS LIKE 'PolarDB_px%';
-- 检查锁等待
SELECT * FROM sys.schema_table_lock_waits;

-- PG版检查
SELECT * FROM pg_stat_activity WHERE state='active' AND now()-query_start > interval '1 minute';
```

## 巡检报告模板

```markdown
# PolarDB 巡检报告
**集群:** xxx | **引擎:** MySQL 8.0 | **架构:** 1主2只读 | **巡检时间:** 2026-03-30

## 集群状态
| 节点 | 角色 | 连接数 | CPU | 内存 | 状态 |
|------|------|--------|-----|------|------|
| 主节点 | Primary | | | | ✅ |
| 只读1 | Replica | | | | ✅ |

## 存储使用
| 指标 | 值 | 状态 |
|------|-----|------|
| 总存储 | | ✅/⚠️ |
| 备份存储 | | |

## 性能指标
## 只读节点延迟
## 异常与建议
```
