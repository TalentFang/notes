# 1.安装
```bash
# 安装 MySQL 服务器
sudo apt install mysql-server -y
# 启动 MySQL 服务
sudo systemctl start mysql
sudo systemctl enable mysql
# 运行安全配置脚本
sudo mysql_secure_installation

# 验证
sudo systemctl status mysql
sudo mysql -u root -p
# 输入密码后进入 MySQL 命令行
```

# 2.基本操作
## MySQL 基本操作

## 一、数据库操作

### 查看与切换
```sql
-- 查看所有数据库
SHOW DATABASES;

-- 使用/切换数据库
USE database_name;

-- 查看当前使用的数据库
SELECT DATABASE();
```

### 创建与删除
```sql
-- 创建数据库
CREATE DATABASE db_name;
CREATE DATABASE IF NOT EXISTS db_name;
CREATE DATABASE db_name CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 删除数据库（谨慎！）
DROP DATABASE db_name;
DROP DATABASE IF EXISTS db_name;
```

## 二、表操作

### 查看表结构
```sql
-- 查看当前数据库的所有表
SHOW TABLES;

-- 查看表结构
DESC table_name;
DESCRIBE table_name;

-- 查看建表语句
SHOW CREATE TABLE table_name;
```

### 创建表
```sql
CREATE TABLE table_name (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL,
    age INT DEFAULT 18,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 修改表结构
```sql
-- 添加列
ALTER TABLE table_name ADD COLUMN phone VARCHAR(20);

-- 修改列类型
ALTER TABLE table_name MODIFY COLUMN phone VARCHAR(15);

-- 修改列名和类型
ALTER TABLE table_name CHANGE COLUMN phone mobile VARCHAR(20);

-- 删除列
ALTER TABLE table_name DROP COLUMN mobile;

-- 重命名表
RENAME TABLE old_name TO new_name;
```

### 删除表
```sql
-- 删除表
DROP TABLE table_name;
DROP TABLE IF EXISTS table_name;

-- 清空表数据（保留表结构）
TRUNCATE TABLE table_name;
DELETE FROM table_name;  -- 可回滚，速度慢
```

## 三、数据操作（CRUD）

### 插入数据
```sql
-- 插入单条
INSERT INTO users (username, email, age) VALUES ('john', 'john@email.com', 25);

-- 插入多条
INSERT INTO users (username, email, age) VALUES 
('alice', 'alice@email.com', 22),
('bob', 'bob@email.com', 28);

-- 插入所有列（不指定列名）
INSERT INTO users VALUES (NULL, 'tom', 'tom@email.com', 30, NOW());
```

### 查询数据
```sql
-- 基础查询
SELECT * FROM users;
SELECT username, email FROM users;
SELECT DISTINCT age FROM users;  -- 去重

-- 条件查询
SELECT * FROM users WHERE age > 20;
SELECT * FROM users WHERE username = 'john' AND age = 25;
SELECT * FROM users WHERE age BETWEEN 20 AND 30;
SELECT * FROM users WHERE username IN ('john', 'alice');
SELECT * FROM users WHERE username LIKE 'j%';  -- 以j开头
SELECT * FROM users WHERE email IS NULL;

-- 排序和限制
SELECT * FROM users ORDER BY age DESC, username ASC;
SELECT * FROM users LIMIT 10;
SELECT * FROM users LIMIT 10 OFFSET 20;  -- 分页

-- 聚合查询
SELECT COUNT(*) FROM users;
SELECT AVG(age), MAX(age), MIN(age) FROM users;
SELECT age, COUNT(*) FROM users GROUP BY age;
SELECT age, COUNT(*) FROM users GROUP BY age HAVING COUNT(*) > 1;

-- 连接查询
SELECT u.username, o.order_date 
FROM users u 
INNER JOIN orders o ON u.id = o.user_id;
```

### 更新数据
```sql
-- 更新单条
UPDATE users SET age = 26 WHERE username = 'john';

-- 更新多条
UPDATE users SET age = age + 1 WHERE age < 30;

-- 更新多列
UPDATE users SET email = 'new@email.com', age = 27 WHERE id = 1;

-- 注意：不加WHERE会更新所有行！
UPDATE users SET status = 'active';  -- 危险操作！
```

### 删除数据
```sql
-- 删除指定行
DELETE FROM users WHERE username = 'tom';

-- 删除所有行
DELETE FROM users;  -- 可回滚，速度慢
TRUNCATE TABLE users;  -- 不可回滚，速度快

-- 注意：不加WHERE会删除所有数据！
DELETE FROM users;  -- 危险操作！
```

## 四、常用函数

### 字符串函数
```sql
SELECT CONCAT(first_name, ' ', last_name) FROM users;
SELECT LENGTH(username) FROM users;
SELECT UPPER(email), LOWER(email) FROM users;
SELECT SUBSTRING(username, 1, 3) FROM users;
SELECT REPLACE(email, '@', '[at]') FROM users;
```

### 日期函数
```sql
SELECT NOW(), CURDATE(), CURTIME();
SELECT YEAR(created_at), MONTH(created_at), DAY(created_at) FROM users;
SELECT DATE_ADD(created_at, INTERVAL 7 DAY) FROM users;
SELECT DATEDIFF(NOW(), created_at) FROM users;
```

### 条件函数
```sql
-- IF函数
SELECT username, IF(age >= 18, 'Adult', 'Minor') FROM users;

-- CASE WHEN
SELECT username,
    CASE 
        WHEN age < 18 THEN 'Young'
        WHEN age BETWEEN 18 AND 65 THEN 'Adult'
        ELSE 'Senior'
    END AS age_group
FROM users;

-- COALESCE（返回第一个非NULL值）
SELECT COALESCE(phone, 'No Phone') FROM users;
```

## 五、索引操作

```sql
-- 创建索引
CREATE INDEX idx_username ON users(username);
CREATE UNIQUE INDEX idx_email ON users(email);

-- 查看索引
SHOW INDEX FROM users;

-- 删除索引
DROP INDEX idx_username ON users;
```

## 六、用户和权限

```sql
-- 创建用户
CREATE USER 'newuser'@'localhost' IDENTIFIED BY 'password';

-- 授予权限
GRANT ALL PRIVILEGES ON db_name.* TO 'newuser'@'localhost';
GRANT SELECT, INSERT ON db_name.* TO 'readonly'@'localhost';

-- 查看权限
SHOW GRANTS FOR 'newuser'@'localhost';

-- 撤销权限
REVOKE ALL PRIVILEGES ON db_name.* FROM 'newuser'@'localhost';

-- 删除用户
DROP USER 'newuser'@'localhost';

-- 刷新权限
FLUSH PRIVILEGES;
```

## 七、实用技巧

### 查看数据库状态
```sql
-- 查看MySQL版本
SELECT VERSION();

-- 查看当前用户
SELECT USER();

-- 查看所有进程
SHOW PROCESSLIST;

-- 查看数据库大小
SELECT table_schema AS 'Database', 
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.tables
GROUP BY table_schema;
```

### 备份与恢复
```bash
# 备份数据库
mysqldump -u root -p db_name > backup.sql

# 备份所有数据库
mysqldump -u root -p --all-databases > all_backup.sql

# 恢复数据库
mysql -u root -p db_name < backup.sql
```

### 常用快捷键（MySQL命令行）
```sql
\c      -- 取消当前输入
\q      -- 退出MySQL
\G      -- 垂直显示结果（用于宽表）
\s      -- 显示服务器状态
\! ls   -- 执行系统命令
```

### 安全注意事项

1. **备份数据**：执行DELETE、UPDATE、DROP前先备份
2. **使用事务**：重要操作先用事务测试
   ```sql
   START TRANSACTION;
   -- 执行操作
   ROLLBACK;  -- 测试后回滚
   -- 或 COMMIT;  -- 确认后提交
   ```

3. **限制查询**：使用LIMIT避免返回大量数据
4. **避免SELECT ***：只查询需要的列
