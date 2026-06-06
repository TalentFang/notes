# Docker 常用命令

## 镜像操作

### 拉取与推送
```bash
# 拉取镜像
docker pull nginx:latest

# 推送镜像
docker push myimage:latest

# 拉取私有仓库镜像
docker pull registry.example.com/myimage:v1
```

### 镜像管理
```bash
# 列出本地镜像
docker images
docker images -a

# 删除镜像
docker rmi nginx:latest
docker rmi $(docker images -q)  # 删除所有

# 构建镜像
docker build -t myapp:v1 .

# 导出/导入镜像
docker save myimage:latest > myimage.tar
docker load < myimage.tar

# 查看镜像历史
docker history myimage:latest
```

## 容器操作

### 创建与启动
```bash
# 运行容器（交互式）
docker run -it ubuntu bash

# 运行容器（守护态）
docker run -d --name mynginx nginx

# 启动/停止/重启容器
docker start mynginx
docker stop mynginx
docker restart mynginx

# 创建但不启动
docker create --name myapp myapp:latest
```

### 常用参数
```bash
# 端口映射
docker run -d -p 8080:80 nginx

# 卷挂载
docker run -d -v /host/data:/container/data nginx

# 环境变量
docker run -d -e MYSQL_ROOT_PASSWORD=secret mysql

# 资源限制
docker run -d --memory=512m --cpus=0.5 nginx

# 容器重命名
docker rename oldname newname
```

### 进入与退出
```bash
# 进入运行中的容器
docker exec -it mynginx bash

# 退出（不停止容器）
exit

# 查看容器日志
docker logs -f mynginx
docker logs --tail 100 mynginx
```

### 容器管理
```bash
# 列出运行中的容器
docker ps

# 列出所有容器（包括已停止）
docker ps -a

# 删除容器
docker rm mynginx
docker rm $(docker ps -aq)  # 删除所有

# 查看容器详情
docker inspect mynginx

# 查看容器资源使用
docker stats

# 查看容器进程
docker top mynginx

# 复制文件
docker cp host.txt mynginx:/container/path.txt
docker cp mynginx:/container/path.txt host.txt
```

## Dockerfile

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 3000
ENV NODE_ENV=production
CMD ["node", "server.js"]
```

### 构建命令
```bash
# 构建（带标签）
docker build -t myapp:v1 .

# 构建时传入变量
docker build --build-arg VERSION=1.0 -t myapp:v1 .

# 多阶段构建
docker build -t myapp:latest .
```

## 数据卷

```bash
# 创建卷
docker volume create mydata

# 列出卷
docker volume ls

# 查看卷详情
docker volume inspect mydata

# 删除未使用的卷
docker volume prune

# 使用卷
docker run -d -v mydata:/data nginx
```

## 网络

```bash
# 创建网络
docker network create mynet

# 列出网络
docker network ls

# 连接容器到网络
docker network connect mynet mynginx

# 断开连接
docker network disconnect mynet mynginx

# 删除网络
docker network rm mynet

# 容器网络模式
# bridge（默认）: --network bridge
# host: --network host
# none: --network none
# 自定义: --network mynet
```

## docker-compose

### 基础命令
```bash
# 启动
docker-compose up -d

# 停止
docker-compose down

# 查看日志
docker-compose logs -f

# 重启
docker-compose restart

# 列出服务
docker-compose ps
```

### 常用参数
```bash
# 指定文件
docker-compose -f docker-compose.prod.yml up -d

# 重新构建
docker-compose up -d --build

# 扩展服务
docker-compose up -d --scale web=3

# 清理资源
docker-compose down -v --rmi all
```

## 清理命令

```bash
# 删除已停止容器
docker container prune

# 删除 dangling 镜像
docker image prune

# 删除所有未使用镜像
docker image prune -a

# 删除已停止容器、未使用网络、 dangling 镜像
docker system prune

# 完全清理（包括卷）
docker system prune --volumes
```

## 常用场景

### 运行 MySQL
```bash
docker run -d \
  --name mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=myapp \
  -v mysql_data:/var/lib/mysql \
  mysql:8
```

### 运行 Redis
```bash
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v redis_data:/data \
  redis:7 \
  redis-server --appendonly yes
```

### 快速测试镜像
```bash
docker run --rm \
  -it \
  --entrypoint sh \
  myimage:latest
```

## 调试

```bash
# 检查容器健康状态
docker inspect --format='{{json .State.Health}}' mynginx

# 查看网络
docker network inspect bridge

# 追踪系统调用（需要特权）
docker run --rm --privileged --pid=host justincormack/nsenter1

# 比较两个镜像
docker diff myimage:v1 myimage:v2
```

## Docker Hub

```bash
# 登录
docker login

# 登出
docker logout

# 搜索镜像
docker search nginx

# 查看镜像标签
curl https://registry.hub.docker.com/v2/repositories/library/nginx/tags
```