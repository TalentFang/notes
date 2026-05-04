我来详细介绍如何编写 Ansible Role，从创建到高级用法。

---

## 一、创建 Role 的两种方式

### 方式1：使用 ansible-galaxy 初始化（推荐）

```bash
# 创建 Role 目录结构
ansible-galaxy init myrole

# 指定路径创建
ansible-galaxy init roles/myrole
```

生成的标准结构：
```
myrole/
├── README.md          # 使用说明
├── defaults/          # 默认变量
│   └── main.yml
├── files/             # 静态文件
├── handlers/          # 处理器
│   └── main.yml
├── meta/              # 元数据
│   └── main.yml
├── tasks/             # 主要任务
│   └── main.yml
├── templates/         # Jinja2 模板
├── tests/             # 测试文件
│   ├── inventory
│   └── test.yml
└── vars/              # 角色变量
    └── main.yml
```

### 方式2：手动创建

```bash
mkdir -p roles/myrole/{defaults,vars,tasks,handlers,templates,files,meta,tests}
touch roles/myrole/{defaults,vars,tasks,handlers,meta}/main.yml
```

---

## 二、各目录详解与编写示例

| 目录/文件               | 必要性          | 用途        |
| ------------------- | ------------ | --------- |
| `tasks/main.yml`    | ⭐⭐⭐ **必须**   | 唯一必须文件    |
| `defaults/main.yml` | ⭐⭐⭐ **强烈建议** | 定义可配置参数   |
| `handlers/main.yml` | ⭐⭐ 常用        | 服务重启等触发操作 |
| `templates/`        | ⭐⭐ 常用        | 动态配置文件    |
| `vars/main.yml`     | ⭐ 偶尔用        | 固定值、系统判断  |
| `files/`            | ⭐ 偶尔用        | 静态文件传输    |
| `meta/main.yml`     | ⭐ 偶尔用        | 依赖其他 role |
| `library/`          | ⭐ 很少用        | 自定义模块     |
| `tests/`            | ⭐ 开发时用       | 本地测试      |
| 其他 `*_plugins/`     | ⭐ 很少用        | 高级扩展      |


### 2.1 tasks/main.yml - 核心任务

**基础结构：**
```yaml
---
# roles/nginx/tasks/main.yml

# 1. 变量验证
- name: 验证必要变量
  assert:
    that:
      - nginx_port is defined
      - nginx_port | int > 0
      - nginx_port | int < 65536
    fail_msg: "nginx_port 必须是 1-65535 之间的整数"
    success_msg: "变量验证通过"

# 2. 包含子任务文件（按功能拆分）
- name: 导入安装任务
  import_tasks: install.yml
  tags: ['nginx', 'install']

- name: 导入配置任务
  import_tasks: configure.yml
  tags: ['nginx', 'config']

- name: 导入服务任务
  import_tasks: service.yml
  tags: ['nginx', 'service']

# 或使用 include_tasks（动态，支持条件）
- name: 导入站点配置
  include_tasks: vhosts.yml
  when: nginx_vhosts | length > 0
```

**子任务文件示例：**

`tasks/install.yml`
```yaml
---
- name: 安装 Nginx（Debian/Ubuntu）
  apt:
    name: nginx
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: 安装 Nginx（RHEL/CentOS）
  yum:
    name: nginx
    state: present
  when: ansible_os_family == "RedHat"

- name: 安装额外模块
  apt:
    name:
      - nginx-extras
      - libnginx-mod-http-geoip2
    state: present
  when: nginx_install_extras | bool
```

`tasks/configure.yml`
```yaml
---
- name: 创建配置目录
  file:
    path: /etc/nginx/conf.d
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: 部署主配置文件
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    validate: 'nginx -t -c %s'    # 验证配置语法
  notify: reload nginx

- name: 删除默认站点
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  when: nginx_remove_default_vhost | bool

- name: 部署站点配置
  template:
    src: vhost.conf.j2
    dest: "/etc/nginx/sites-available/{{ item.server_name }}.conf"
    mode: '0644'
  loop: "{{ nginx_vhosts }}"
  loop_control:
    label: "{{ item.server_name }}"
  notify: reload nginx

- name: 启用站点
  file:
    src: "/etc/nginx/sites-available/{{ item.server_name }}.conf"
    dest: "/etc/nginx/sites-enabled/{{ item.server_name }}.conf"
    state: link
  loop: "{{ nginx_vhosts }}"
  notify: reload nginx
```

`tasks/service.yml`
```yaml
---
- name: 启动并启用 Nginx
  service:
    name: nginx
    state: started
    enabled: yes

- name: 检查 Nginx 响应
  uri:
    url: "http://localhost:{{ nginx_port }}"
    status_code: 200
  register: nginx_check
  retries: 5
  delay: 2
  until: nginx_check.status == 200
```

---

### 2.2 defaults/main.yml - 默认变量

**特点：**
- 优先级最低，最容易被覆盖
- 用于定义"安全默认值"
- 所有变量都应有默认值，避免未定义错误

```yaml
---
# roles/nginx/defaults/main.yml

# 基本配置
nginx_port: 80
nginx_user: "www-data"
nginx_worker_processes: "auto"
nginx_worker_connections: 4096

# 性能调优
nginx_client_max_body_size: "64m"
nginx_keepalive_timeout: 65
nginx_gzip: true

# 功能开关
nginx_install_extras: false
nginx_remove_default_vhost: true

# 站点列表（空列表表示不配置站点）
nginx_vhosts: []
# 示例：
# nginx_vhosts:
#   - server_name: example.com
#     root: /var/www/example
#     index: index.html
#     ssl: false
#   - server_name: api.example.com
#     root: /var/www/api
#     ssl: true
#     ssl_cert: /etc/ssl/certs/api.crt
#     ssl_key: /etc/ssl/private/api.key

# 日志配置
nginx_access_log: "/var/log/nginx/access.log"
nginx_error_log: "/var/log/nginx/error.log"

# 安全头
nginx_security_headers: true
nginx_hide_version: true
```

---

### 2.3 vars/main.yml - 角色变量

**特点：**
- 优先级高于 defaults
- 通常用于定义不希望被轻易覆盖的变量
- 可包含动态计算的值

```yaml
---
# roles/nginx/vars/main.yml

# 固定值，不建议覆盖
nginx_package_name: "nginx"
nginx_service_name: "nginx"
nginx_config_path: "/etc/nginx"

# 根据系统动态设置
nginx_user: "{{ 'www-data' if ansible_os_family == 'Debian' else 'nginx' }}"

# 依赖包列表
nginx_dependencies:
  Debian:
    - nginx
    - openssl
    - ca-certificates
  RedHat:
    - nginx
    - openssl
    - ca-certificates

nginx_packages: "{{ nginx_dependencies[ansible_os_family] }}"

# 内部使用的常量
__nginx_config_test_command: "nginx -t"
```

---

### 2.4 handlers/main.yml - 处理器

```yaml
---
# roles/nginx/handlers/main.yml

- name: start nginx
  service:
    name: nginx
    state: started

- name: stop nginx
  service:
    name: nginx
    state: stopped

- name: restart nginx
  service:
    name: nginx
    state: restarted
  listen: "restart web server"    # 多个 notify 可以监听同一事件

- name: reload nginx
  service:
    name: nginx
    state: reloaded
  listen: "reload web server"

- name: validate nginx config
  command: nginx -t
  changed_when: false             # 验证不改变状态
```

**触发方式：**
```yaml
# 在 tasks 中 notify
- name: 修改配置
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: 
    - reload nginx
    # 或
    - "reload web server"
```

---

### 2.5 templates/ - Jinja2 模板

`templates/nginx.conf.j2`
```jinja2
# {{ ansible_managed }} - 自动生成，不要手动编辑
# 生成时间: {{ ansible_date_time.iso8601 }}

user {{ nginx_user }};
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
    use epoll;
    multi_accept on;
}

http {
    # 基础设置
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout {{ nginx_keepalive_timeout }};
    types_hash_max_size 2048;

    # 服务器令牌
    {% if nginx_hide_version %}
    server_tokens off;
    {% else %}
    server_tokens on;
    {% endif %}

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log {{ nginx_access_log }} main;
    error_log {{ nginx_error_log }};

    # Gzip 压缩
    {% if nginx_gzip %}
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json 
               application/javascript application/rss+xml 
               application/atom+xml image/svg+xml;
    {% endif %}

    # 安全头
    {% if nginx_security_headers %}
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    {% endif %}

    # 包含站点配置
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

`templates/vhost.conf.j2`
```jinja2
server {
    listen {{ nginx_port }};
    server_name {{ item.server_name }};
    root {{ item.root }};
    index {{ item.index | default('index.html index.htm') }};

    # 访问日志
    access_log /var/log/nginx/{{ item.server_name }}-access.log;
    error_log /var/log/nginx/{{ item.server_name }}-error.log;

    {% if item.ssl | default(false) %}
    # SSL 配置
    listen 443 ssl;
    ssl_certificate {{ item.ssl_cert }};
    ssl_certificate_key {{ item.ssl_key }};
    ssl_protocols TLSv1.2 TLSv1.3;
    {% endif %}

    # 位置块
    location / {
        try_files $uri $uri/ =404;
    }

    # 自定义位置块
    {% for location in item.locations | default([]) %}
    location {{ location.path }} {
        {{ location.config | indent(8) }}
    }
    {% endfor %}
}
```

---

### 2.6 files/ - 静态文件

放置不需要模板化的静态文件：

```
files/
├── index.html          # 默认首页
├── 50x.html           # 错误页面
├── favicon.ico
└── healthcheck.html   # 健康检查页
```

使用方式：
```yaml
- name: 部署静态文件
  copy:
    src: "{{ item }}"
    dest: "/var/www/html/{{ item | basename }}"
    mode: '0644'
  loop:
    - index.html
    - 50x.html
    - favicon.ico
```

---

### 2.7 meta/main.yml - 元数据

```yaml
---
# roles/nginx/meta/main.yml

galaxy_info:
  role_name: nginx
  author: your_name
  description: 安装和配置 Nginx Web 服务器
  company: Your Company
  license: MIT
  min_ansible_version: "2.9"
  
  # 支持的平台
  platforms:
    - name: Ubuntu
      versions:
        - focal      # 20.04
        - jammy      # 22.04
    - name: EL        # Enterprise Linux
      versions:
        - "8"
        - "9"
    - name: Debian
      versions:
        - bullseye   # 11
        - bookworm   # 12
  
  # Galaxy 标签
  galaxy_tags:
    - web
    - nginx
    - server
    - http
    - proxy

# 依赖关系
dependencies:
  # 简单依赖
  - role: common
  
  # 带参数的依赖
  - role: firewall
    vars:
      firewall_ports:
        - { port: "{{ nginx_port }}", proto: tcp }
        - { port: 443, proto: tcp }
    when: configure_firewall | default(true) | bool
  
  # 条件依赖
  - role: logrotate
    vars:
      logrotate_scripts:
        - name: nginx
          path: /var/log/nginx/*.log
          options:
            - daily
            - rotate 14
            - compress
    when: nginx_logrotate | default(true) | bool
```

---

### 2.8 tests/ - 测试文件

`tests/inventory`
```ini
[local]
localhost ansible_connection=local
```

`tests/test.yml`
```yaml
---
- name: 测试 Nginx Role
  hosts: local
  become: yes
  gather_facts: yes
  
  vars:
    nginx_port: 8080
    nginx_vhosts:
      - server_name: test.local
        root: /tmp/testwww
        index: index.html
  
  pre_tasks:
    - name: 创建测试目录
      file:
        path: /tmp/testwww
        state: directory
  
    - name: 创建测试页面
      copy:
        content: "<h1>Test Page</h1>"
        dest: /tmp/testwww/index.html
  
  roles:
    - role: nginx
  
  post_tasks:
    - name: 测试 Nginx 响应
      uri:
        url: "http://localhost:8080"
        status_code: 200
      register: result
    
    - name: 显示测试结果
      debug:
        msg: "Nginx 测试成功！状态码: {{ result.status }}"
```

运行测试：
```bash
cd roles/nginx/tests
ansible-playbook test.yml
```

---

## 三、高级用法

### 3.1 Role 参数验证

`tasks/main.yml` 开头添加验证：
```yaml
---
- name: 验证输入参数
  assert:
    that:
      - nginx_port is defined
      - nginx_port | int > 0
      - nginx_vhosts is iterable
    fail_msg: "参数验证失败，请检查变量定义"
  tags: ['always']

- name: 验证每个站点配置
  assert:
    that:
      - item.server_name is defined
      - item.root is defined
    fail_msg: "站点 {{ item }} 缺少必要配置"
  loop: "{{ nginx_vhosts }}"
  loop_control:
    label: "{{ item.server_name | default('undefined') }}"
```

### 3.2 动态包含任务

```yaml
# 根据变量动态选择任务文件
- name: 导入系统特定任务
  include_tasks: "{{ ansible_os_family }}.yml"

# 或
- name: 导入安装方式特定任务
  include_tasks: "install-{{ nginx_install_method }}.yml"
  vars:
    nginx_install_method: "{{ 'source' if nginx_version == 'latest' else 'package' }}"
```

### 3.3 使用 block 和 rescue

```yaml
- name: 安全地修改配置
  block:
    - name: 备份原配置
      copy:
        src: /etc/nginx/nginx.conf
        dest: /etc/nginx/nginx.conf.backup.{{ ansible_date_time.epoch }}
        remote_src: yes
    
    - name: 部署新配置
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: reload nginx
  
  rescue:
    - name: 恢复备份配置
      copy:
        src: "/etc/nginx/nginx.conf.backup.{{ ansible_date_time.epoch }}"
        dest: /etc/nginx/nginx.conf
        remote_src: yes
    
    - name: 报告错误
      fail:
        msg: "配置部署失败，已恢复原配置"
```

### 3.4 注册和使用变量

```yaml
- name: 检查 Nginx 版本
  command: nginx -v
  register: nginx_version_check
  changed_when: false
  ignore_errors: yes

- name: 设置版本事实
  set_fact:
    nginx_installed_version: "{{ nginx_version_check.stderr | regex_search('nginx/([\\d.]+)', '\\1') | first }}"
  when: nginx_version_check.rc == 0

- name: 显示版本信息
  debug:
    msg: "当前 Nginx 版本: {{ nginx_installed_version | default('未安装') }}"
```

---

## 四、在 Playbook 中使用 Role

### 4.1 基础用法

```yaml
---
- name: 部署 Web 服务器
  hosts: webservers
  become: yes
  
  roles:
    - common
    - nginx
    - php
```

### 4.2 带参数和条件

```yaml
---
- name: 多环境部署
  hosts: all
  become: yes
  
  roles:
    - role: common
      tags: ['always']
    
    - role: nginx
      vars:
        nginx_port: 8080
        nginx_worker_processes: 4
      when: "'web' in group_names"
      tags: ['web', 'nginx']
    
    - role: mysql
      vars:
        mysql_root_password: "{{ vault_mysql_root_pass }}"
      when: "'db' in group_names"
      tags: ['db', 'mysql']
```

### 4.3 使用 import_role 和 include_role（Ansible 2.4+）

```yaml
---
- name: 动态 Role 调用
  hosts: webservers
  become: yes
  
  tasks:
    # 静态导入（解析时处理，性能好）
    - name: 导入基础 Role
      import_role:
        name: common
      tags: ['always']
    
    # 动态包含（运行时处理，支持条件）
    - name: 根据条件加载 Web 服务器
      include_role:
        name: "{{ web_server_type }}"    # 变量决定加载 nginx 还是 apache
      vars:
        web_server_type: nginx
      when: install_web_server | bool
    
    # 在任务间调用 Role
    - name: 准备数据
      command: /opt/prepare.sh
    
    - name: 部署应用
      include_role:
        name: app
      vars:
        app_version: "2.1.0"
    
    - name: 验证部署
      uri:
        url: http://localhost/health
```

---

## 五、Role 开发最佳实践

### 5.1 目录组织建议

```
roles/
├── common/              # 系统基础配置（时区、用户、SSH）
├── firewall/            # 防火墙管理
├── nginx/               # Web 服务器
├── php/                 # PHP 环境
├── mysql/               # 数据库
├── redis/               # 缓存
├── supervisor/          # 进程管理
└── app-deploy/          # 应用部署（业务相关）
```

### 5.2 命名规范

| 项目 | 规范 | 示例 |
|------|------|------|
| Role 名称 | 小写，下划线分隔 | `nginx_ssl`, `mysql_backup` |
| 变量名 | role名_变量名 | `nginx_port`, `mysql_root_password` |
| 标签 | 功能分类 | `install`, `config`, `service` |
| 任务名 | 动词+对象 | "Install Nginx", "Configure vhost" |

### 5.3 幂等性保证

```yaml
# ✅ 好的做法：使用模块而非命令
- name: 启动服务（幂等）
  service:
    name: nginx
    state: started
    enabled: yes

# ❌ 避免：非幂等
- name: 启动服务（非幂等）
  command: systemctl start nginx

# ✅ 使用 creates/removes
- name: 编译安装（幂等）
  command: make install
  args:
    chdir: /tmp/source
    creates: /usr/local/bin/app    # 文件存在则跳过
```

### 5.4 文档规范

**README.md 模板：**
```markdown
# Ansible Role: Nginx

安装和配置 Nginx Web 服务器。

## 要求

- Ansible 2.9+
- Ubuntu 20.04/22.04 或 CentOS 8/9

## 角色变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| nginx_port | 80 | 监听端口 |
| nginx_user | www-data | 运行用户 |
| nginx_vhosts | [] | 站点列表 |

## 依赖

- common
- firewall（可选）

## 示例 Playbook

```yaml
- hosts: webservers
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
```
# 六、role各个文件夹之间的关系

我来详细解释 Ansible Role 中不同文件夹的关系，以及 YAML 文件的处理机制。

## 文件夹关系与优先级

```
变量优先级（低到高）：
defaults/ < vars/ < playbook 中的 vars < 命令行 -e

执行顺序：
meta/ → tasks/ → handlers/（被触发时）
```

## 各文件夹 YAML 文件详解

### 1. `tasks/main.yml` — 核心任务

```yaml
---
- name: 安装 Nginx
  apt:
    name: nginx
    state: present

- name: 复制配置文件
  template:
    src: nginx.conf.j2      # 自动去 templates/ 目录找
    dest: /etc/nginx/nginx.conf
  notify: restart nginx     # 触发 handler

- name: 复制静态文件
  copy:
    src: index.html         # 自动去 files/ 目录找
    dest: /var/www/html/
```

**关键点**：
- `template` 模块自动到 `templates/` 找 `.j2` 文件
- `copy` 模块自动到 `files/` 找文件
- `notify` 触发 `handlers/` 中同名任务

---

### 2. `defaults/main.yml` vs `vars/main.yml`

| 特性 | `defaults/` | `vars/` |
|------|-------------|---------|
| 优先级 | 低 | 高 |
| 用途 | 可被覆盖的默认值 | 固定值，不建议覆盖 |
| 场景 | 端口、路径等配置 | 系统判断的固定值 |

```yaml
# defaults/main.yml - 用户可覆盖
nginx_port: 80
nginx_user: www-data

# vars/main.yml - 通常由系统决定
nginx_config_path: "{% if ansible_os_family == 'Debian' %}/etc/nginx{% else %}/etc/nginx{% endif %}"
```

---

### 3. `handlers/main.yml` — 触发器

```yaml
---
- name: restart nginx      # 必须和 notify 中的名字完全匹配
  service:
    name: nginx
    state: restarted

- name: reload nginx
  service:
    name: nginx
    state: reloaded
```

**触发机制**：
```yaml
# 在 tasks 中
- name: 修改配置
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: restart nginx    # ← 配置变化时才触发
```

---

### 4. `templates/` 与 `files/` 的区别

| 目录 | 用途 | 模块 |
|------|------|------|
| `files/` | 静态文件，原样复制 | `copy`, `script` |
| `templates/` | 动态模板，变量替换 | `template` |

```yaml
# templates/nginx.conf.j2 - 使用 Jinja2 语法
server {
    listen {{ nginx_port }};           # 引用变量
    server_name {{ ansible_hostname }}; # 内置变量
    
    {% if enable_ssl %}
    ssl_certificate {{ cert_path }};   # 条件判断
    {% endif %}
}
```

---

### 5. `meta/main.yml` — 依赖声明

```yaml
---
dependencies:
  - role: common          # 先执行 common role
  - role: firewall
    vars:                 # 传递特定变量
      open_ports:
        - 80
        - 443
```

**执行顺序**：`meta` 中的依赖 → 当前 role 的 `tasks`

---

## 文件拆分与引用

### 大型 role 如何拆分

```
tasks/
├── main.yml          # 入口，通过 include_tasks 拆分
├── install.yml       # 安装步骤
├── configure.yml     # 配置步骤
└── service.yml       # 服务管理
```

```yaml
# tasks/main.yml
---
- include_tasks: install.yml
- include_tasks: configure.yml
- include_tasks: service.yml
```

### 条件导入

```yaml
# tasks/main.yml
---
- include_tasks: setup-Debian.yml
  when: ansible_os_family == "Debian"

- include_tasks: setup-RedHat.yml
  when: ansible_os_family == "RedHat"
```

---

## 完整数据流向图

```
playbook 调用 role
    ↓
meta/main.yml（依赖的 role 先执行）
    ↓
defaults/main.yml（加载默认变量）
    ↓
vars/main.yml（加载固定变量，覆盖 defaults）
    ↓
tasks/main.yml（开始执行）
    ├── template 模块 → 查找 templates/*.j2
    ├── copy 模块 → 查找 files/*
    └── notify → 触发 handlers/main.yml
```

---

## 实用建议

| 场景 | 推荐做法 |
|------|---------|
| 用户可改的配置 | 放 `defaults/` |
| 系统判断的值 | 放 `vars/` 或用 `set_fact` |
| 多平台支持 | `tasks/` 拆分 + `when` 条件 |
| 配置文件复杂 | 用 `templates/` + Jinja2 |
| 二进制/静态资源 | 放 `files/` |
| 服务需要重启 | 用 `handlers` + `notify` |


# License

MIT
```

---

## 六、调试技巧

```bash
# 语法检查
ansible-playbook site.yml --syntax-check

# 干运行（检查变更）
ansible-playbook site.yml --check --diff

# 仅运行特定 Role（通过标签）
ansible-playbook site.yml --tags "nginx"

# 查看 Role 执行顺序
ansible-playbook site.yml --list-tasks

# 详细调试
ansible-playbook site.yml -vvv

# 仅针对特定主机测试 Role
ansible-playbook site.yml --limit testserver --check
```

掌握这些编写规范后，你就能创建出**可维护、可复用、可测试**的高质量 Ansible Role。