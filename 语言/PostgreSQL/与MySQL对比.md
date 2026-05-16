# PostgreSQL 与 MySQL 对比

## 架构差异

| 方面 | PostgreSQL | MySQL |
|---|---|---|
| 架构模型 | 进程模式（每个连接一个进程） | 线程模式（每个连接一个线程） |
| 默认端口 | 5432 | 3306 |
| 存储引擎 | 单一（统一存储引擎） | 多种（InnoDB, MyISAM 等） |
| 默认字符集 | UTF8 | latin1 |

## SQL 语法差异

### 自增主键
```sql
-- PostgreSQL
CREATE TABLE users (
    id SERIAL PRIMARY KEY
);

-- MySQL
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY
);
```

### 字符串引号
```sql
-- PostgreSQL：单引号用于值，双引号用于标识符
SELECT "name" FROM "users" WHERE 'active' = 'yes';

-- MySQL：反引号用于标识符，单引号用于值
SELECT `name` FROM `users` WHERE 'active' = 'yes';
```

### 分页
```sql
-- PostgreSQL
SELECT * FROM users LIMIT 10 OFFSET 20;

-- MySQL
SELECT * FROM users LIMIT 20, 10;
```

### 插入返回值
```sql
-- PostgreSQL: RETURNING
INSERT INTO users (name) VALUES ('张三') RETURNING id;

-- MySQL: LAST_INSERT_ID()
INSERT INTO users (name) VALUES ('张三');
SELECT LAST_INSERT_ID();
```

### 模糊匹配
```sql
-- PostgreSQL: ILIKE（大小写不敏感）
SELECT * FROM users WHERE name ILIKE '%张三%';

-- MySQL: LIKE（默认大小写不敏感）
SELECT * FROM users WHERE name LIKE '%张三%';
```

## 功能差异

| 功能 | PostgreSQL | MySQL |
|---|---|---|
| 事务 | ACID，完全支持 | ACID，InnoDB 完整支持 |
| MVCC | 原生支持 | InnoDB 支持 |
| 隔离级别 | 4种（读已提交、可重复读、串行化） | 4种 |
| JSON 支持 | JSON/JSONB 原生 | JSON（5.7+） |
| 数组类型 | 原生支持 | 不支持（需用 JSON） |
| 范围类型 | 原生支持 | 不支持 |
| GIS | PostGIS 扩展 | Spatial 数据类型 |
| 全文搜索 | 内置 | MyISAM 支持 |
| 外键约束 | 完整支持 | 完整支持 |
| 触发器 | BEFORE/AFTER/INSTEAD OF | BEFORE/AFTER |
| 存储过程 | 支持 | 支持 |
| 自定义类型 | 支持 | 不支持 |
| 继承表 | 支持 | 不支持 |

## 性能对比

| 场景 | PostgreSQL | MySQL |
|---|---|---|
| 复杂查询 | 更优 | 一般 |
| 简单查询 | 稍慢 | 更优 |
| 高并发写入 | 一般 | 更优（InnoDB） |
| 大量读 | 一般 | 更优 |
| 事务处理 | 更优 | 良好 |
| 大数据量 | 更优 | 需要优化 |

## 索引类型

| 索引 | PostgreSQL | MySQL |
|---|---|---|
| B-tree | 支持 | 支持 |
| Hash | 支持 | 支持（Memory引擎） |
| GiST | 支持（地理信息） | 不支持 |
| GIN | 支持（倒排索引） | 不支持 |
| SP-GiST | 支持 | 不支持 |
| 表达式索引 | 支持 | 部分支持 |
| 部分索引 | 支持 | 不支持 |

## 管理对比

| 操作 | PostgreSQL | MySQL |
|---|---|---|
| 启动 | `pg_ctl start` | `service mysql start` |
| 连接 | `psql -U user -d db` | `mysql -u user -p db` |
| 备份 | `pg_dump` | `mysqldump` |
| 主从复制 | 流复制 | 二进制日志 |
| 集群方案 | Citus、PGCluster | MySQL Cluster |

## 适用场景

| 场景 | 推荐 |
|---|---|
| 复杂业务逻辑 | PostgreSQL |
| 高并发简单写入 | MySQL |
| GIS 应用 | PostgreSQL + PostGIS |
| JSON 文档存储 | PostgreSQL (JSONB) |
| 事务型应用 | 两者皆可 |
| 数据分析 | PostgreSQL |
| Web 小型应用 | MySQL |

## 迁移注意

1. **字符转义**：PostgreSQL 用 `''`，MySQL 用 `\'`
2. **自增类型**：PostgreSQL 用 `SERIAL`，MySQL 用 `AUTO_INCREMENT`
3. **标识符引号**：PostgreSQL 用双引号，MySQL 用反引号
4. **分页语法**：PostgreSQL 用 `LIMIT x OFFSET y`
5. **布尔值**：PostgreSQL 用 `TRUE/FALSE`，MySQL 用 `1/0`