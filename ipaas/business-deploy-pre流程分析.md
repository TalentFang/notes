# business-deploy-pre 流程分析

> 源码路径: `roles/business-deploy-pre/`
> 分析日期: 2026-06-04

## 一、概述

`business-deploy-pre` 角色负责在目标机器上生成以下关键脚本：

| 脚本 | 渲染后路径 | 用途 |
|------|-----------|------|
| `deploy.sh.j2` | `/data/asap/deploy.sh` | **首次全量部署** |
| `deploy_scale.sh.j2` | `/data/asap/deploy_scale.sh` | **扩容/缩容/修改IP** |
| `deploy_upper.sh.j2` | `/data/asap/deploy_upper.sh` | **行业版本批量部署** |
| `deploy_check_ini.sh.j2` | `/data/asap/deploy_check_ini.sh` | **健康检查配置初始化** |
| `db_version_init.sh.j2` | `/data/asap/db_version_init.sh` | **微服务版本信息初始化** |

`vars/main.yml` 定义了一个关键变量：
```yaml
SERVER_VIP: "{{ groups['vip'][0] if groups['vip']|length > 0 else groups['all'][0] }}"
```

---

## 二、扩容操作流程 (scale-component)

### 调用方式

```bash
sh /data/asap/deploy_scale.sh -t scale-component <组件名> -a
# 例如：
sh /data/asap/deploy_scale.sh -t scale-component es -a
```

### 参数说明

| 参数 | 含义 |
|------|------|
| `-t scale-component` | 任务类型：组件扩容 |
| `-a` | 开启 pre + post + sql 全部标志 |
| `-c <服务名>` | 可选，指定单个服务 |
| `-i <目标IP>` | 可选，指定执行目标 IP |
| `-s` | 可选，仅执行 SQL |

第二个位置参数 `<组件名>` 会被追加到 `task_type` 后面，形成 `scale-component/es`，最终读取的脚本目录为：

```
/data/asap/deploy/<服务名>/sh/scale/scale-component/<组件名>/sh/
```

### 执行流程

```
deploy_scale.sh -t scale-component <组件名> -a
│
├── 1. 遍历 /data/asap/deploy 下所有服务（或指定的 -c 服务）
│
├── 2. deploy_one_service(服务名)
│   │
│   ├── [pre_flag=1] 执行 pre.sh
│   │   └── /data/asap/deploy/<服务名>/sh/scale/scale-component/<组件名>/sh/pre.sh
│   │
│   ├── [sql_flag=1] deploy_sql(服务名)
│   │   ├── 检查 scale/scale-component/<组件名>/sql/ 目录
│   │   ├── ansible-playbook → 02.business-service-sql.yml (MySQL)
│   │   ├── ansible-playbook → 02.business-service-sql-dm8.yml (达梦)
│   │   └── ansible-playbook → 02.business-service-sql-vastbase.yml (VastBase)
│   │
│   ├── deploy_sh(服务名) — 遍历 sh/ 下的子目录
│   │   ├── 目录命名规则：<groupname>_<opertype>，如 es_all、es_one
│   │   ├── deploy_sh_pre_post() → 通过 ansible-playbook 执行：
│   │   │   └── 08.business-sh-scale-deploy.yml
│   │   │       └── roles/business-sh-scale-deploy
│   │   │           ├── rsync 同步脚本到目标机器
│   │   │           └── 执行 sh <shname>.sh
│   │   └── 如果指定了 -i <IP>，limit 到该 IP 执行
│   │
│   └── [post_flag=1] 执行 post.sh
│       └── /data/asap/deploy/<服务名>/sh/scale/scale-component/<组件名>/sh/post.sh
│
└── 3. 结束
```

### 业务服务目录结构

```
/data/asap/deploy/<服务名>/
├── sh/
│   └── scale/
│       └── scale-component/
│           └── <组件名>/
│               ├── sh/
│               │   ├── pre.sh                          # 全局前置脚本
│               │   ├── post.sh                         # 全局后置脚本
│               │   ├── <groupname>_all/scale.sh        # 在所有节点执行
│               │   └── <groupname>_one/scale.sh        # 在首个节点执行
│               └── sql/
│                   ├── *.sql                           # MySQL SQL
│                   ├── dm8/*.sql                       # 达梦 SQL
│                   └── vastbase/*.sql                  # VastBase SQL
└── ...
```

---

## 三、缩容操作流程

缩容与扩容共用 `deploy_scale.sh`，任务类型不同：

### 调用方式

```bash
sh /data/asap/deploy_scale.sh -t refresh-config -a
```

### 参数说明

`-t` 参数仅接受三种值：
- `refresh-config` — 刷新配置
- `scale-component` — 扩容组件
- `modify-ip` — 修改 IP

缩容实际通过 **`refresh-config`** 类型执行，读取的脚本目录为：

```
/data/asap/deploy/<服务名>/sh/scale/refresh-config/sh/
```

### 执行流程

与扩容流程一致，区别在于：
1. `task_type = refresh-config`
2. 读取的 sh/sql 目录不同：`sh/scale/refresh-config/`
3. 所有服务按字母升序依次执行

---

## 四、修改 IP 操作流程 (modify-ip)

### 调用方式

```bash
sh /data/asap/deploy_scale.sh -t modify-ip -i <目标IP> -a
# 例如：
sh /data/asap/deploy_scale.sh -t modify-ip -i 10.20.30.40 -a
```

### 关键差异：`-i` 参数

当指定 `-i <目标IP>` 时：
1. `ip_flag=1`，`dest_ip=<目标IP>`
2. `deploy_sh_pre_post()` 函数中，ansible-playbook 的 `--limit` **限制为该目标 IP**，而非整个组
3. 意味着 **shell 脚本只在指定的新 IP 机器上执行**

### 执行流程

```
deploy_scale.sh -t modify-ip -i <目标IP> -a
│
├── 1. 遍历所有服务
│
├── 2. deploy_one_service(服务名)
│   │
│   ├── [pre_flag=1] 执行 pre.sh
│   │
│   ├── [sql_flag=1] deploy_sql(服务名)
│   │   └── 读取 sh/scale/modify-ip/sql/ 下的 SQL
│   │
│   ├── deploy_sh(服务名)
│   │   └── deploy_sh_pre_post()
│   │       ├── ip_flag=1 时：
│   │       │   ansible-playbook --limit <目标IP> ...
│   │       └── ip_flag=0 时：
│   │           ansible-playbook --limit <组名> ...
│   │
│   └── [post_flag=1] 执行 post.sh
│
└── 3. 结束
```

### VIP 变量

`deploy_scale.sh.j2` 中定义了 `REPLACE_IP="{{ SERVER_VIP }}"`，来源于 `vars/main.yml`：
```yaml
SERVER_VIP: "{% if groups['vip']|length > 0 %}{{ groups['vip'][0] }}{% else %}{{ groups['all'][0] }}{% endif %}"
```
此变量在模板渲染时被替换为目标集群的 VIP 地址。

---

## 五、deploy.sh 首次全量部署流程

### 调用方式

```bash
sh /data/asap/deploy.sh -p /data/asap/deploy -a
# -p: 服务路径
# -a: 开启 pre + post + sql + docker 全部标志
```

### 参数说明

| 参数 | 含义 |
|------|------|
| `-a` | 全部开启：pre + post + sql + docker |
| `-c <服务名>` | 指定单个服务 |
| `-p <路径>` | 服务路径（默认 `/data/asap/deploy`） |
| `-d` | 仅执行 Docker 构建 |
| `-s` | 仅执行 SQL |
| `-b pre/post/all` | 仅执行前置/后置 shell |
| `-f` | 强制部署（忽略 optional 文件） |

### 执行流程

```
deploy.sh -p /data/asap/deploy -a
│
├── 1. 获取 Docker registry 地址
│   └── 从 /etc/docker/daemon.json 读取 insecure-registries
│
├── 2. main() — 遍历 /data/asap/deploy 下所有服务（字母升序）
│
├── 3. deploy_one_service(服务名)
│   │
│   ├── [pre_flag=1] deploy_sh(服务名, "pre")
│   │   ├── 执行 sh/pre/pre.sh
│   │   ├── 遍历 sh/pre/ 下 <groupname>_<opertype> 子目录
│   │   │   ├── deploy_sh_pre_post() → sh_pre_post_tool.sh
│   │   └── 执行 sh/pre/post.sh
│   │
│   ├── [sql_flag=1] deploy_sql(服务名)
│   │   ├── ansible-playbook → 02.business-service-sql.yml
│   │   ├── ansible-playbook → 02.business-service-sql-dm8.yml (如有)
│   │   ├── ansible-playbook → 02.business-service-sql-vastbase.yml (如有)
│   │   ├── ansible-playbook → 02.business-service-sql-ck.yml (如有)
│   │   └── 执行产品专属 SQL：sql/<产品名>/*.sql
│   │
│   ├── [docker_flag=1] deploy_dockerfile(服务名)
│   │   ├── 检查 raw-image/Dockerfile 是否存在
│   │   ├── 替换基础镜像为私有 registry 地址
│   │   ├── 达梦8特殊处理：拷贝 dm_svc.conf
│   │   ├── docker build → 生成镜像
│   │   ├── docker push → 推送到 registry
│   │   ├── 修改 k8s.yaml 中的 image 字段
│   │   ├── 集群场景：副本数 < 2 时设为 2，添加节点互斥
│   │   ├── kubectl apply -f k8s.yaml → 部署 Pod
│   │   ├── 部署 Ingress
│   │   ├── 写入 k8s_service.txt / k8s_ingress.txt
│   │   └── 写入 k8s_check.txt（用于健康检查）
│   │
│   └── [post_flag=1] deploy_sh(服务名, "post")
│       └── 与 pre 流程一致
│
└── 4. 结束
```

### deploy_upper.sh 行业版本批量部署

```bash
sh /data/asap/deploy_upper.sh
```

该脚本遍历 `/data/asap/deploy-*`（所有行业版本目录），对每个目录执行：
1. `sh /data/asap/deploy.sh -p <行业目录> -a` — 全量部署
2. `sh /data/asap/db_version_init.sh <行业目录>` — 初始化版本信息

---

## 六、upgrade_deploy.sh 版本升级流程

### 调用方式

```bash
sh /data/maxs-ops/tools/upgrade_deploy.sh -p <升级包路径> -c <服务名> -v <版本号> -a
```

### 参数说明

| 参数 | 含义 |
|------|------|
| `-p <路径>` | 升级包路径 |
| `-c <服务名>` | 目标服务名 |
| `-v <版本号>` | 升级目标版本 |
| `-a` | 全部开启 |
| `-s` | 仅执行 SQL |

### 执行流程

```
upgrade_deploy.sh -p <路径> -c <服务名> -v <版本号> -a
│
├── 验证必填参数（path/version/servicename）
│
├── deploy_one_service(服务名, 版本号)
│   │
│   ├── [pre_flag=1] deploy_sh(服务名, 版本号, "pre")
│   │   └── <路径>/<服务名>/<版本号>/sh/pre/
│   │       ├── pre.sh
│   │       ├── <groupname>_one/ 或 <groupname>_all/ 脚本
│   │       └── post.sh
│   │
│   ├── [sql_flag=1] deploy_sql(服务名, 版本号)
│   │   └── <路径>/<服务名>/<版本号>/sql/
│   │       ├── *.sql
│   │       └── <产品名>/*.sql
│   │
│   └── [post_flag=1] deploy_sh(服务名, 版本号, "post")
│       └── <路径>/<服务名>/<版本号>/sh/post/
│
└── 结束
```

### 与 deploy.sh 的区别

| 对比项 | deploy.sh | upgrade_deploy.sh |
|--------|-----------|-------------------|
| 用途 | 首次全量部署 | 增量版本升级 |
| Docker 构建 | 有 | 无 |
| K8s 部署 | 有 | 无 |
| 服务目录 | `/data/asap/deploy/<服务名>/` | `<路径>/<服务名>/<版本号>/` |
| 外部调用 | 由 deploy role 调用 | 由部署编排器（如 Jenkins）调用 |

---

## 七、健康检查配置初始化 (deploy_check_ini.sh)

在 `deploy_check_ini.sh.j2` 中定义了 4 个函数，用于向 `/data/asap/maxs_status_check/conf.ini` 注入检查项：

| 函数 | 数据源 | 注入位置 |
|------|--------|---------|
| `add_k8s_check()` | `k8s_check.txt` | `[k8s]` 节后 |
| `add_systemd_check()` | `systemd_check.txt` | `asap_node_exporter` 行后 |
| `add_k8s_service_check()` | `k8s_service.txt` | `k8s_service` 行后 |
| `add_k8s_ingress_check()` | `k8s_ingress.txt` | `k8s_ingress` 行后 |

这些文件在 `deploy.sh` 的 Docker 部署阶段被动态写入：
- `deploy.sh` → `write_service()` → 写入 `k8s_service.txt`
- `deploy.sh` → `write_ingress()` → 写入 `k8s_ingress.txt`
- `deploy.sh` → 追加到 `k8s_check.txt`

---

## 八、Ansible Playbook 调用关系

```
playbooks/business/00.business-pre.yml
  └── roles/business-deploy-pre  ← 生成所有脚本

playbooks/business/01.business.yml
  └── roles/business-deploy      ← 执行 deploy.sh + db_version_init.sh + deploy_upper.sh + deploy_check_ini.sh

playbooks/business/08.business-sh-scale-deploy.yml
  └── roles/business-sh-scale-deploy  ← 扩缩容脚本执行
```

---

## 九、各操作对比总结

| 操作 | 入口脚本 | 任务类型 | 脚本目录 | 是否构建镜像 |
|------|---------|---------|---------|-------------|
| 首次部署 | `deploy.sh` | - | `sh/pre/`, `sh/post/` | 是 |
| 扩容 | `deploy_scale.sh -t scale-component` | `scale-component` | `sh/scale/scale-component/<组件>/` | 否 |
| 缩容 | `deploy_scale.sh -t refresh-config` | `refresh-config` | `sh/scale/refresh-config/` | 否 |
| 修改 IP | `deploy_scale.sh -t modify-ip -i <IP>` | `modify-ip` | `sh/scale/modify-ip/` | 否 |
| 版本升级 | `upgrade_deploy.sh` | - | `<版本号>/sh/`, `<版本号>/sql/` | 否 |
| 行业部署 | `deploy_upper.sh` | - | 调用 deploy.sh | 是 |
