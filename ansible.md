 Ansible 是一个强大的自动化运维工具。

## Ansible 概述

**Ansible** 是由 Red Hat 开发的开源 IT 自动化工具，主要用于：
- **配置管理**（Configuration Management）
- **应用部署**（Application Deployment）
- **任务编排**（Orchestration）
- **持续交付**（Continuous Delivery）

### 核心特点

| 特性 | 说明 |
|------|------|
| **无代理架构** | 基于 SSH 工作，无需在被管节点安装客户端 |
| **简单易用** | 使用 YAML 语法（Playbook），学习曲线低 |
| **幂等性** | 多次执行结果一致，不会重复操作 |
| **模块化设计** | 丰富的模块库（3000+），覆盖各类运维场景 |
| **可扩展性** | 支持自定义模块和插件 |

---

## 核心组件

### 1. 控制节点（Control Node）
运行 Ansible 命令的主机，需安装 Python 和 Ansible。

### 2. 被管节点（Managed Nodes）
被 Ansible 管理的服务器，只需支持 SSH 和 Python。

### 3. Inventory（主机清单）

参考：[[ansible inventory怎么解读]]
定义被管节点的列表和分组，支持 INI 或 YAML 格式：

```ini
# /etc/ansible/hosts (INI 格式)
[webservers]
web1.example.com ansible_ssh_host=192.168.1.10
web2.example.com ansible_ssh_host=192.168.1.11

[dbservers]
db1.example.com ansible_ssh_host=192.168.1.20

[production:children]
webservers
dbservers
```

### 4. Playbook（剧本）

参考：[[ansible playbook]]
YAML 格式的自动化脚本，定义任务流程：

```yaml
---
- name: 部署 Web 服务器
  hosts: webservers
  become: yes  # 使用 sudo 提权
  
  vars:
    http_port: 80
    max_clients: 200
  
  tasks:
    - name: 安装 Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes
    
    - name: 启动 Nginx 服务
      service:
        name: nginx
        state: started
        enabled: yes
    
    - name: 部署配置文件
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: restart nginx
  
  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

### 5. Modules（模块）
模块用法：[[ansible 模块用法]]
Ansible 的功能单元，常用模块包括：

| 模块                  | 用途           |
| ------------------- | ------------ |
| `yum`/`apt`         | 包管理          |
| `copy`              | 复制文件         |
| `template`          | 模板渲染（Jinja2） |
| `file`              | 文件/目录管理      |
| `service`/`systemd` | 服务管理         |
| `user`/`group`      | 用户管理         |
| `command`/`shell`   | 执行命令         |
| `cron`              | 定时任务         |
| `mount`             | 挂载管理         |
| `docker_container`  | Docker 容器管理  |
]]

---

## 常用命令

[[ansible 常用命令行]]

例如：
```bash
# 测试连通性
ansible all -m ping

# 执行 ad-hoc 命令
ansible webservers -m command -a "uptime"

# 安装软件包
ansible dbservers -m apt -a "name=mysql-server state=present" -b

# 复制文件
ansible all -m copy -a "src=/local/file dest=/remote/file mode=644"

# 运行 Playbook
ansible-playbook site.yml

# 语法检查
ansible-playbook site.yml --syntax-check

# 模拟运行（不实际执行）
ansible-playbook site.yml --check --diff

# 指定主机运行
ansible-playbook deploy.yml --limit webservers

# 查看详细输出
ansible-playbook site.yml -vvv
```

---

## 高级特性

### 1. Roles（角色）
参考：[[ansible roles怎么写]]
[[ansible role和playbook的关系]]

结构化组织 Playbook，实现代码复用：

```
roles/
├── common/          # 角色名称
│   ├── tasks/       # 任务列表 (main.yml)
│   ├── handlers/    # 处理器 (main.yml)
│   ├── templates/   # Jinja2 模板
│   ├── files/       # 静态文件
│   ├── vars/        # 变量 (main.yml)
│   ├── defaults/    # 默认变量 (main.yml)
│   ├── meta/        # 元数据 (main.yml)
│   └── tests/       # 测试文件
└── nginx/
    └── ...
```


使用角色：
```yaml
- hosts: webservers
  roles:
    - common
    - nginx
```

### 2. Variables（变量）
支持多种变量来源（优先级由低到高）：
- 命令行 (`-e`)
- 角色默认变量
- Inventory 变量
- Playbook 变量
- 注册变量 (`register`)
- 额外变量文件

### 3. Vault（加密）
保护敏感数据（密码、密钥等）：

```bash
# 创建加密文件
ansible-vault create secrets.yml

# 编辑加密文件
ansible-vault edit secrets.yml

# 运行包含加密文件的 Playbook
ansible-playbook site.yml --ask-vault-pass
# 或
ansible-playbook site.yml --vault-password-file .vault_pass
```

### 4. Dynamic Inventory（动态清单）
从云平台（AWS、Azure、阿里云等）动态获取主机列表：

```bash
# 使用 AWS EC2 动态清单
ansible-playbook -i aws_ec2.yml site.yml
```

### 5. Ansible Galaxy
官方角色仓库，共享和下载社区角色：

```bash
# 安装角色
ansible-galaxy install geerlingguy.nginx

# 从 requirements.yml 安装
ansible-galaxy install -r requirements.yml
```

---

## 典型应用场景

### 1. 批量系统初始化
```yaml
- name: 系统初始化
  hosts: all
  tasks:
    - name: 配置 SSH
      template:
        src: sshd_config.j2
        dest: /etc/ssh/sshd_config
    
    - name: 创建运维用户
      user:
        name: ops
        groups: sudo
        shell: /bin/bash
    
    - name: 配置防火墙
      ufw:
        rule: allow
        port: "{{ item }}"
      loop:
        - 22
        - 80
        - 443
```

### 2. 应用部署流水线
```yaml
- name: 部署 Java 应用
  hosts: appservers
  vars:
    app_version: "2.1.0"
    deploy_path: "/opt/myapp"
  
  tasks:
    - name: 拉取代码
      git:
        repo: https://github.com/example/app.git
        dest: "{{ deploy_path }}"
        version: "v{{ app_version }}"
    
    - name: 构建应用
      command: mvn clean package
      args:
        chdir: "{{ deploy_path }}"
    
    - name: 部署到 Tomcat
      copy:
        src: "{{ deploy_path }}/target/app.war"
        dest: /var/lib/tomcat9/webapps/
      notify: restart tomcat
```

### 3. 数据库备份
```yaml
- name: MySQL 定时备份
  hosts: dbservers
  tasks:
    - name: 创建备份目录
      file:
        path: /backup/mysql
        state: directory
        mode: '0755'
    
    - name: 执行备份
      mysql_db:
        state: dump
        name: all
        target: "/backup/mysql/backup_{{ ansible_date_time.date }}.sql.gz"
    
    - name: 清理旧备份（保留7天）
      shell: find /backup/mysql -name "*.sql.gz" -mtime +7 -delete
```

---

## Ansible vs 其他工具

| 特性 | Ansible | Puppet | Chef | SaltStack |
|------|---------|--------|------|-----------|
| **架构** | 无代理（SSH） | 有代理 | 有代理 | 有/无代理 |
| **语法** | YAML | DSL | Ruby | YAML/Python |
| **学习曲线** | 低 | 中 | 高 | 中 |
| **执行方式** | 推送（Push） | 拉取（Pull） | 拉取（Pull） | 混合 |
| **适用规模** | 中小规模 | 大规模 | 大规模 | 大规模 |
| **实时执行** | 支持 | 不支持 | 不支持 | 支持 |

---

## 最佳实践

1. **使用版本控制**：将 Playbook 纳入 Git 管理
2. **幂等性设计**：确保任务可重复执行
3. **最小权限原则**：使用 `become` 精确控制提权
4. **敏感数据加密**：使用 Ansible Vault 保护密钥
5. **模块化组织**：使用 Roles 拆分复杂任务
6. **测试先行**：使用 Molecule 进行角色测试
7. **文档注释**：为复杂 Playbook 添加注释

---

## 安装 Ansible

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install ansible

# CentOS/RHEL
sudo yum install epel-release
sudo yum install ansible

# macOS
brew install ansible

# Python pip
pip install ansible
```

Ansible 凭借其简洁的设计和强大的功能，已成为 DevOps 领域最受欢迎的自动化工具之一，特别适合快速上手和中小规模的运维自动化场景。