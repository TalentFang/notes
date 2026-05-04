 我来详细解释 Role 和 Playbook 的关系，这是 Ansible 中非常重要的架构概念。

---

## 一、核心关系图解

```
┌─────────────────────────────────────────────────────────────┐
│                    Playbook（剧本）                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - name: 部署 Web 服务器                             │    │
│  │    hosts: webservers                                 │    │
│  │    become: yes                                       │    │
│  │                                                      │    │
│  │    roles:              ◄── 调用 Role                │    │
│  │      - common          ◄── 系统基础配置             │    │
│  │      - nginx           ◄── Web 服务器               │    │
│  │      - php             ◄── PHP 环境                 │    │
│  │      - mysql           ◄── 数据库                   │    │
│  │                                                      │    │
│  │    tasks:              ◄── Playbook 专属任务        │    │
│  │      - name: 额外配置                                │    │
│  │        ...                                           │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │              Role（角色）                │
        │  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
        │  │  tasks  │  │templates│  │  files  │ │
        │  │  main.yml│  │  .j2   │  │ 静态文件 │ │
        │  └─────────┘  └─────────┘  └─────────┘ │
        │  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
        │  │  vars   │  │ defaults│  │ handlers│ │
        │  │ main.yml│  │ main.yml│  │ main.yml│ │
        │  └─────────┘  └─────────┘  └─────────┘ │
        │  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
        │  │  meta   │  │  tests  │  │  README │ │
        │  │ main.yml│  │         │  │  .md    │ │
        │  └─────────┘  └─────────┘  └─────────┘ │
        └─────────────────────────────────────────┘
```

**一句话总结**：
> **Playbook 是导演，Role 是演员。Playbook 负责编排场景（hosts、顺序、变量），Role 负责具体的表演（安装、配置、服务管理）。**

---

## 二、详细对比

| 特性 | Playbook | Role |
|------|----------|------|
| **定位** | 编排和入口 | 功能封装和复用 |
| **内容** | 调用 Role、定义 hosts、变量、执行顺序 | 具体的 tasks、templates、handlers 等 |
| **复用性** | 通常针对特定场景 | 高度可复用，可在多个 Playbook 中使用 |
| **目录结构** | 单个 YAML 文件 | 标准化的目录结构（8个目录） |
| **变量作用域** | 全局或 Play 级别 | Role 内部或可通过参数传入 |
| **依赖管理** | 通过 Role 的 meta 实现 | 在 meta/main.yml 中声明依赖 |

---

## 三、实际使用示例

### 4.1 基础用法

**Playbook 文件：site.yml**
```yaml
---
- name: 部署 Web 应用
  hosts: webservers
  become: yes
  
  roles:
    - common        # 先执行 common Role
    - nginx         # 再执行 nginx Role
    - php
```

**Role 调用顺序**：按列表顺序依次执行

---

### 4.2 带参数的 Role

```yaml
---
- name: 部署多站点
  hosts: webservers
  become: yes
  
  roles:
    # 第一次调用，参数不同
    - role: nginx
      vars:
        nginx_port: 8080
        server_name: "api.example.com"
        document_root: "/var/www/api"
    
    # 第二次调用，不同参数
    - role: nginx
      vars:
        nginx_port: 8081
        server_name: "admin.example.com"
        document_root: "/var/www/admin"
```

---

### 4.3 条件执行 Role

```yaml
---
- name: 智能部署
  hosts: all
  become: yes
  
  roles:
    - role: common
      tags: ['always']        # 始终执行
    
    - role: nginx
      when: "'webservers' in group_names"  # 仅在 webservers 组执行
    
    - role: mysql
      when: "'dbservers' in group_names"
    
    - role: monitoring
      tags: ['monitoring']    # 仅当指定 --tags monitoring 时执行
```

---

### 4.4 Role 与 Playbook 任务混合

```yaml
---
- name: 复杂部署流程
  hosts: webservers
  become: yes
  
  pre_tasks:                   # Role 之前执行
    - name: 开始部署通知
      debug:
        msg: "开始部署到 {{ inventory_hostname }}"
  
  roles:
    - common
    - nginx
  
  tasks:                       # Role 之后执行
    - name: 部署应用代码
      git:
        repo: https://github.com/app/repo.git
        dest: /var/www/app
    
    - name: 检查服务状态
      uri:
        url: http://localhost/health
        status_code: 200
  
  post_tasks:                  # 最后执行
    - name: 部署完成通知
      debug:
        msg: "部署完成！"
```

**执行顺序**：
1. `pre_tasks` → `handlers`（如果被 notify）
2. `roles`（按顺序）
3. `tasks` → `handlers`（如果被 notify）
4. `post_tasks` → `handlers`（如果被 notify）

---

## 六、Ansible Galaxy 与 Role 分享

```bash
# 从 Galaxy 安装 Role
ansible-galaxy install geerlingguy.nginx

# 安装到项目目录
ansible-galaxy install geerlingguy.nginx -p ./roles/

# 使用 requirements.yml 批量安装
ansible-galaxy install -r requirements.yml
```

**requirements.yml**
```yaml
---
roles:
  # 从 Galaxy
  - name: geerlingguy.nginx
    version: 3.1.0
  
  # 从 Git
  - name: custom-role
    src: https://github.com/org/custom-role.git
    version: main
  
  # 从本地
  - name: local-role
    src: file:///path/to/role.tar.gz
```

---

## 七、两种写法对比

### 1.直接在playbook写task
```yaml
# 原来在 playbook 中
- name: 部署 Nginx
  hosts: web
  tasks:
    - name: 安装 Nginx
      apt: name=nginx state=present
    
    - name: 启动服务
      service: name=nginx state=started
```

### 2.playbook与role组合
* role文件
```yaml
# 变成 Role 后
# roles/nginx/tasks/main.yml
- name: 安装 Nginx
  apt: 
    name: nginx 
    state: present

- name: 启动服务
  service: 
    name: nginx 
    state: started
```
* playbook文件
```yaml
# Playbook 变得简洁
- name: 部署 Web
  hosts: web
  roles:
    - nginx
```


---

## 八、总结

```
Playbook (site.yml)            ← 编排层
    │
    ├── hosts: webservers      ← 目标主机
    │
    ├── vars:                  ← 全局变量
    │
    ├── pre_tasks:             ← 前置任务
    │
    ├── roles:                  ← 调用 Role
    │     ├── common           → roles/common/tasks/main.yml
    │     ├── nginx            → roles/nginx/tasks/main.yml
    │     │                       ├── 自动加载 defaults/main.yml
    │     │                       ├── 自动加载 vars/main.yml
    │     │                       ├── template → 找 templates/nginx.conf.j2
    │     │                       └── notify → 触发 handlers/main.yml
    │     └── firewall
    │
    └── tasks:                 ← 后置特定任务
```

**简单记忆**：
- **Playbook** = 电影剧本（场景、角色安排、流程）
- **Role** = 演员（具体表演、服装、道具）

通过这种分层设计，Ansible 实现了**高内聚、低耦合**的自动化架构，既保证了功能的原子化和复用性，又保留了足够的灵活性来应对复杂场景。