---
name: sql-writing
description: SQL编写技能。当用户需要编写SQL查询、SQL优化、复杂查询设计、多表JOIN、窗口函数、CTE、子查询、数据透视、递归查询、SQL转换、SQL兼容性处理（MySQL/PostgreSQL/Oracle/SQL Server/达梦）、SQL性能分析、执行计划解读时使用。涵盖所有主流数据库的SQL语法差异和最佳实践。
---

# SQL 编写技能

## 编写原则

1. **可读性优先**：合理缩进，有意义的别名，避免过度嵌套
2. **性能意识**：先想执行计划再写SQL，避免全表扫描
3. **兼容性**：标注数据库方言差异，提供多版本写法

## 基础规范

### 命名与格式
```sql
-- 表别名用有意义的缩写
SELECT o.order_id, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- 关键字大写，字段小写（推荐）
SELECT order_id, customer_name
FROM orders
WHERE create_time > '2026-01-01'
  AND status = 1;
```

### 条件过滤
```sql
-- 避免函数导致索引失效
-- ❌ WHERE DATE(create_time) = '2026-03-30'
-- ✅ WHERE create_time >= '2026-03-30' AND create_time < '2026-03-31'

-- ✅ 使用 EXISTS 替代 IN（大数据量时）
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM order_items i WHERE i.order_id = o.id);

-- LIKE前置通配符避免
-- ❌ WHERE name LIKE '%abc%'    -- 全表扫描
-- ✅ WHERE name LIKE 'abc%'    -- 可用索引
```

## 窗口函数

### 排名
```sql
-- ROW_NUMBER: 无并列，连续编号
-- RANK: 并列跳跃
-- DENSE_RANK: 并列不跳跃
SELECT emp_name, dept, salary,
       ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) rn,
       RANK() OVER (PARTITION BY dept ORDER BY salary DESC) rnk,
       DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) drnk
FROM employees;
```

### 聚合窗口
```sql
-- 累计求和
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date) running_total
FROM orders;

-- 移动平均
SELECT date, value,
       AVG(value) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) ma_7d
FROM metrics;

-- 取首尾
SELECT dept, emp_name, salary,
       FIRST_VALUE(salary) OVER (PARTITION BY dept ORDER BY hire_date) first_salary,
       LAST_VALUE(salary) OVER (PARTITION BY dept ORDER BY hire_date
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) last_salary,
       LAG(salary, 1) OVER (PARTITION BY dept ORDER BY hire_date) prev_salary,
       LEAD(salary, 1) OVER (PARTITION BY dept ORDER BY hire_date) next_salary
FROM employees;

-- 去重取最新
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_time DESC) rn
    FROM user_logins
) t WHERE rn = 1;
```

## CTE（公用表表达式）

```sql
-- 基础CTE
WITH dept_stats AS (
    SELECT dept_id, COUNT(*) emp_count, AVG(salary) avg_salary
    FROM employees GROUP BY dept_id
)
SELECT d.dept_name, s.emp_count, s.avg_salary
FROM dept_stats s
JOIN departments d ON s.dept_id = d.id;

-- 递归CTE（组织架构树）
WITH RECURSIVE org_tree AS (
    SELECT id, name, parent_id, 1 AS level
    FROM departments WHERE parent_id IS NULL
    UNION ALL
    SELECT d.id, d.name, d.parent_id, o.level + 1
    FROM departments d
    JOIN org_tree o ON d.parent_id = o.id
)
SELECT * FROM org_tree ORDER BY level, name;

-- SQL Server递归写法（不加RECURSIVE）
WITH org_tree AS (
    SELECT id, name, parent_id, 1 AS level
    FROM departments WHERE parent_id IS NULL
    UNION ALL
    SELECT d.id, d.name, d.parent_id, o.level + 1
    FROM departments d JOIN org_tree o ON d.parent_id = o.id
)
SELECT * FROM org_tree;
```

## 数据透视

### MySQL
```sql
-- 行转列（GROUP_CONCAT + CASE WHEN）
SELECT create_date,
    SUM(CASE WHEN status=1 THEN 1 ELSE 0 END) pending,
    SUM(CASE WHEN status=2 THEN 1 ELSE 0 END) processing,
    SUM(CASE WHEN status=3 THEN 1 ELSE 0 END) completed
FROM orders GROUP BY create_date;

-- MySQL 8.0 PIVOT: 不原生支持，用条件聚合
-- Oracle 11g+: PIVOT关键字
SELECT * FROM (
    SELECT dept, score, name FROM students
) PIVOT (AVG(score) FOR dept IN ('数学' AS math, '英语' AS english, '语文' AS chinese));
```

### PostgreSQL
```sql
-- crosstab（需扩展 tablefunc）
CREATE EXTENSION tablefunc;
SELECT * FROM crosstab(
    'SELECT student, subject, score FROM scores ORDER BY 1,2',
    'SELECT DISTINCT subject FROM scores ORDER BY 1'
) AS ct(student VARCHAR, math INT, english INT, chinese INT);
-- 或用 FILTER
SELECT student,
    AVG(score) FILTER (WHERE subject='数学') math,
    AVG(score) FILTER (WHERE subject='英语') english
FROM scores GROUP BY student;
```

### SQL Server
```sql
SELECT * FROM (
    SELECT dept, score, name FROM students
) s PIVOT (AVG(score) FOR dept IN ([数学],[英语],[语文])) p;
-- UNPIVOT
SELECT name, subject, score FROM (
    SELECT name, math, english, chinese FROM student_scores
) s UNPIVOT (score FOR subject IN (math, english, chinese)) u;
```

## 多表 JOIN

```sql
-- INNER JOIN: 只返回匹配行
-- LEFT JOIN: 保留左表全部行
-- RIGHT JOIN: 保留右表全部行
-- FULL OUTER JOIN: 全部保留（MySQL不支持，用UNION模拟）
-- CROSS JOIN: 笛卡尔积

-- 复杂JOIN示例：找到每个部门最高薪的员工
SELECT d.dept_name, e.emp_name, e.salary
FROM employees e
JOIN departments d ON e.dept_id = d.id
WHERE e.salary = (SELECT MAX(salary) FROM employees WHERE dept_id = e.dept_id);

-- 等价写法（推荐，更高效）
SELECT d.dept_name, e.emp_name, e.salary
FROM (
    SELECT dept_id, emp_name, salary,
           ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) rn
    FROM employees
) e JOIN departments d ON e.dept_id = d.id
WHERE e.rn = 1;
```

## 数据库方言速查

| 功能 | MySQL | PostgreSQL | Oracle | SQL Server | 达梦 |
|------|-------|-----------|--------|-----------|------|
| 分页 | LIMIT n OFFSET m | LIMIT m OFFSET n | ROWNUM / FETCH | OFFSET m ROWS FETCH NEXT n | LIMIT m OFFSET n |
| 字符串拼接 | CONCAT(a,b) | a \|\| b | a \|\| b | a + b | a \|\| b |
| 当前时间 | NOW() | NOW() | SYSDATE | GETDATE() | SYSDATE |
| IF | IF(cond,a,b) | CASE WHEN | CASE WHEN | CASE WHEN | CASE WHEN |
| 自增 | AUTO_INCREMENT | SERIAL/IDENTITY | SEQUENCE | IDENTITY | IDENTITY |
| 模糊匹配 | REGEXP | ~ 正则 | REGEXP_LIKE | LIKE | REGEXP |
| 递归CTE | WITH RECURSIVE | WITH RECURSIVE | WITH (CONNECT BY) | WITH | WITH RECURSIVE |
| MERGE | INSERT ON DUPLICATE KEY | INSERT ON CONFLICT | MERGE INTO | MERGE | MERGE INTO |
| 窗口函数 | 8.0+ | 全支持 | 全支持 | 全支持 | 全支持 |
| JSON | JSON函数 | JSONB | JSON | JSON | JSON |

## 性能优化检查清单

1. **EXPLAIN 先行**：写完SQL先看执行计划
2. **索引覆盖**：查询字段尽量被索引覆盖
3. **避免 SELECT ***：只查需要的列
4. **JOIN顺序**：小表驱动大表
5. **子查询转JOIN**：相关子查询改写为JOIN
6. **分批处理**：大数据量操作分批（LIMIT + OFFSET循环）
7. **避免隐式转换**：类型不匹配会导致全表扫描

## SQL模板

### 分页
```sql
-- MySQL/PG/达梦
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 20;
-- Oracle
SELECT * FROM (
    SELECT a.*, ROWNUM rn FROM (SELECT * FROM orders ORDER BY id) a WHERE ROWNUM <= 30
) WHERE rn > 20;
-- Oracle 12c+
SELECT * FROM orders ORDER BY id OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
-- SQL Server
SELECT * FROM orders ORDER BY id OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

### 批量插入
```sql
-- MySQL
INSERT INTO orders (id, name, amount) VALUES (1,'a',100),(2,'b',200);
-- Oracle
INSERT ALL INTO orders VALUES (1,'a',100) INTO orders VALUES (2,'b',200) SELECT 1 FROM DUAL;
-- SQL Server
INSERT INTO orders VALUES (1,'a',100),(2,'b',200);
```

### UPSERT
```sql
-- MySQL
INSERT INTO kv (key, value) VALUES ('k1','v1') ON DUPLICATE KEY UPDATE value='v1';
-- PG
INSERT INTO kv (key, value) VALUES ('k1','v1') ON CONFLICT (key) DO UPDATE SET value='v1';
-- Oracle
MERGE INTO kv t USING (SELECT 'k1' key, 'v1' val FROM DUAL) s
ON (t.key = s.key) WHEN MATCHED THEN UPDATE SET value=s.val WHEN NOT MATCHED THEN INSERT (key,value) VALUES (s.key,s.val);
-- SQL Server
MERGE INTO kv t USING (VALUES ('k1','v1')) s(key,val)
ON t.key=s.key WHEN MATCHED THEN UPDATE SET value=s.val WHEN NOT MATCHED THEN INSERT (key,value) VALUES (s.key,s.val);
```
