```shell
# 更新源
sudo apt update
sudo apt upgrade -y
```

## 1.安装
```bash
# 安装 Redis
sudo apt install redis-server -y

# 启动 Redis 服务
sudo systemctl start redis-server
sudo systemctl enable redis-server

# 验证安装
redis-cli ping
# 应该返回 PONG
```

## 2.基本配置
```bash
# 编辑配置文件
sudo nano /etc/redis/redis.conf

# 如需允许远程连接，修改：
# bind 127.0.0.1 -> bind 0.0.0.0
# 设置密码（推荐）：
# requirepass your_password

# 重启 Redis
sudo systemctl restart redis-server
```