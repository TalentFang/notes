 我来详细介绍 Ansible 常用模块的使用方法，包括语法、参数和实际示例。

---

## 一、文件操作类模块

### 1.1 `copy` - 复制文件

**用途**：将控制节点的文件复制到被管节点

```yaml
# 基础用法
- name: 复制配置文件
  copy:
    src: /local/nginx.conf      # 控制节点源文件
    dest: /etc/nginx/nginx.conf # 目标路径
    owner: root
    group: root
    mode: '0644'                # 权限，建议用字符串避免八进制问题
    backup: yes                 # 备份原文件（加时间戳）

# 复制目录（递归）
- name: 复制整个目录
  copy:
    src: /local/configs/        # 注意：末尾有/则复制内容，无/则复制目录本身
    dest: /remote/configs/
    directory_mode: '0755'

# 使用 content 直接写入内容
- name: 创建 index.html
  copy:
    content: |
      <html>
        <body>
          <h1>Welcome to {{ ansible_hostname }}</h1>
        </body>
      </html>
    dest: /var/www/html/index.html
    mode: '0644'

# 验证文件后再复制
- name: 条件复制
  copy:
    src: app.conf
    dest: /etc/app/config.conf
    validate: 'app --test-config %s'  # 验证命令，%s 代表临时文件路径
```

**关键参数**：
| 参数 | 说明 |
|------|------|
| `src` | 源文件路径（相对或绝对） |
| `dest` | 目标路径 |
| `mode` | 权限（0644、0755、u+rwx 等） |
| `owner`/`group` | 属主/属组 |
| `backup` | 是否备份原文件 |
| `force` | 是否强制覆盖（默认 yes） |
| `content` | 直接指定内容，替代 src |

---

### 1.2 `template` - 模板渲染

**用途**：使用 Jinja2 模板引擎渲染文件

```yaml
# 基础用法（模板文件以 .j2 结尾）
- name: 部署 Nginx 配置
  template:
    src: nginx.conf.j2          # 模板文件位于 templates/ 目录或相对路径
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: yes
  notify: reload nginx

# 模板示例 (templates/nginx.conf.j2)
# user {{ nginx_user }};
# worker_processes {{ ansible_processor_vcpus }};
# server {
#     listen {{ http_port | default(80) }};
#     server_name {{ server_name }};
#     {% if ssl_enabled %}
#     listen 443 ssl;
#     {% endif %}
# }

# 使用变量和过滤器
- name: 生成 hosts 文件
  template:
    src: hosts.j2
    dest: /etc/hosts
  vars:
    local_ip: "{{ ansible_default_ipv4.address }}"

# 模板中常用语法：
# {{ variable }}           - 变量插值
# {{ var | default('x') }} - 默认值
# {% if condition %}       - 条件判断
# {% for item in list %}   - 循环
# {# comment #}            - 注释
```

---

### 1.3 `file` - 文件/目录管理

**用途**：管理文件属性、创建删除目录、创建符号链接

```yaml
# 创建目录
- name: 创建应用目录
  file:
    path: /opt/myapp
    state: directory          # 目录
    owner: appuser
    group: appgroup
    mode: '0755'
    recurse: yes              # 递归设置权限

# 创建文件（空文件）
- name: 创建空日志文件
  file:
    path: /var/log/myapp/app.log
    state: touch
    owner: appuser
    mode: '0644'

# 创建软链接
- name: 创建符号链接
  file:
    src: /opt/myapp/current
    dest: /opt/myapp/latest
    state: link
    force: yes                # 强制创建（如果目标存在）

# 创建硬链接
- name: 创建硬链接
  file:
    src: /etc/nginx/nginx.conf
    dest: /root/nginx.conf.bak
    state: hard

# 删除文件/目录
- name: 清理临时文件
  file:
    path: /tmp/old_data
    state: absent             # 删除

# 修改权限（不创建文件）
- name: 修改权限
  file:
    path: /etc/shadow
    mode: '0640'
    owner: root
    group: shadow
```

**state 参数**：
- `file`：确保是普通文件（不存在则失败）
- `directory`：确保是目录
- `touch`：确保文件存在（不存在则创建空文件）
- `link`：符号链接
- `hard`：硬链接
- `absent`：删除

---

### 1.4 `lineinfile` - 行编辑

**用途**：确保文件中某行存在、修改或删除特定行

```yaml
# 确保某行存在（不存在则添加，存在则不变）
- name: 配置 SSH 端口
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?Port '        # 匹配以 Port 开头的行（#可选）
    line: 'Port 2222'         # 替换为这行
    state: present
    backup: yes
  notify: restart sshd

# 删除某行
- name: 禁止 root 登录
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin no'
    state: present

# 在特定行后插入
- name: 添加配置
  lineinfile:
    path: /etc/app/config.conf
    insertafter: '^# Database settings'  # 在此行之后插入
    line: 'max_connections = 100'
    state: present

# 在文件末尾添加（如果不存在）
- name: 添加环境变量
  lineinfile:
    path: /etc/environment
    line: 'JAVA_HOME=/usr/lib/jvm/java-11'
    state: present
    create: yes               # 文件不存在则创建

# 删除匹配行
- name: 删除旧配置
  lineinfile:
    path: /etc/nginx/nginx.conf
    regexp: 'server_name old.example.com'
    state: absent

# 使用 backrefs 进行复杂替换
- name: 修改监听地址
  lineinfile:
    path: /etc/mysql/my.cnf
    regexp: '^(bind-address\s*=\s*)127.0.0.1'
    line: '\g<1>0.0.0.0'      # \g<1> 引用第一个捕获组
    backrefs: yes             # 启用反向引用
```

---

### 1.5 `blockinfile` - 块编辑

**用途**：管理文件中的多行文本块

```yaml
# 添加/更新代码块
- name: 配置 Ansible 托管标记
  blockinfile:
    path: /etc/hosts
    block: |
      # Ansible managed block
      192.168.1.10 web1.local
      192.168.1.11 web2.local
      192.168.1.12 db1.local
    marker: "# {mark} ANSIBLE MANAGED BLOCK"  # 标记行，{mark} 会被替换为 BEGIN/END
    state: present
    backup: yes

# 生成的文件内容：
# # BEGIN ANSIBLE MANAGED BLOCK
# # Ansible managed block
# 192.168.1.10 web1.local
# 192.168.1.11 web2.local
# 192.168.1.12 db1.local
# # END ANSIBLE MANAGED BLOCK

# 插入到特定位置
- name: 在 http 块中添加配置
  blockinfile:
    path: /etc/nginx/nginx.conf
    block: |
      location /api {
          proxy_pass http://localhost:8080;
      }
    insertbefore: '^}$'        # 在最后一个 } 之前插入
    marker: "# {mark} API CONFIG"

# 删除块
- name: 移除旧配置
  blockinfile:
    path: /etc/nginx/nginx.conf
    marker: "# {mark} OLD CONFIG"
    state: absent
```

---

### 1.6 `fetch` - 拉取文件

**用途**：从被管节点拉取文件到控制节点（与 copy 相反）

```yaml
# 拉取日志文件
- name: 收集日志
  fetch:
    src: /var/log/nginx/error.log
    dest: /local/logs/{{ inventory_hostname }}/  # 会自动创建目录结构
    flat: no                    # no：保持目录结构；yes：直接放入 dest

# 拉取多个配置文件
- name: 备份配置文件
  fetch:
    src: "{{ item }}"
    dest: /backup/{{ inventory_hostname }}/
    flat: no
  loop:
    - /etc/nginx/nginx.conf
    - /etc/nginx/sites-enabled/default
    - /etc/php/8.1/fpm/php.ini

# flat=yes 示例（所有文件放同一目录，需确保文件名不冲突）
- name: 收集状态文件
  fetch:
    src: /var/run/app.pid
    dest: /local/pids/{{ inventory_hostname }}.pid
    flat: yes
```

---

### 1.7 `synchronize` - 文件同步（rsync）

**用途**：高效的文件同步，基于 rsync

```yaml
# 基础同步（控制节点 → 被管节点）
- name: 同步网站代码
  synchronize:
    src: /local/webapp/
    dest: /var/www/html/
    rsync_opts:
      - "--exclude=.git"
      - "--exclude=node_modules"
      - "--delete"             # 删除目标端多余文件

# 被管节点 → 控制节点（mode=pull）
- name: 拉取备份数据
  synchronize:
    src: /data/backups/
    dest: /local/backups/{{ inventory_hostname }}/
    mode: pull
    compress: yes

# 被管节点之间同步
- name: 主从同步
  synchronize:
    src: /data/master/
    dest: /data/slave/
    mode: push
  delegate_to: master_server   # 在 master 上执行
```

**注意**：需要安装 rsync，且使用 SSH 密钥认证更方便

---

## 二、包管理类模块

### 2.1 `apt`（Debian/Ubuntu）

```yaml
# 安装单个包
- name: 安装 Nginx
  apt:
    name: nginx
    state: present            # present|absent|latest

# 安装多个包
- name: 安装开发工具
  apt:
    name:
      - build-essential
      - git
      - curl
      - vim
    state: present

# 更新缓存并安装
- name: 安装并更新缓存
  apt:
    name: nginx
    state: present
    update_cache: yes
    cache_valid_time: 3600    # 缓存有效期（秒），避免频繁更新

# 升级所有包
- name: 全面升级系统
  apt:
    upgrade: dist             # safe|full|dist
    update_cache: yes

# 删除包
- name: 卸载旧版本
  apt:
    name: apache2
    state: absent
    purge: yes                # 同时删除配置文件
    autoremove: yes           # 删除不再依赖的包

# 安装 .deb 包
- name: 安装本地 deb 包
  apt:
    deb: /tmp/package.deb
    state: present

# 添加 PPA 并安装
- name: 添加 PHP PPA
  apt_repository:
    repo: ppa:ondrej/php
    state: present
  become: yes

- name: 安装 PHP 8.2
  apt:
    name: php8.2-fpm
    state: present
    update_cache: yes
```

---

### 2.2 `yum` / `dnf`（RHEL/CentOS/Fedora）

```yaml
# 基础安装
- name: 安装 Nginx
  yum:
    name: nginx
    state: present
    # 或 dnf:
    # dnf:
    #   name: nginx
    #   state: present

# 安装多个包
- name: 安装工具包
  yum:
    name:
      - vim
      - wget
      - net-tools
      - epel-release
    state: present

# 从 URL 安装
- name: 安装 RPM 包
  yum:
    name: https://example.com/package.rpm
    state: present

# 启用/禁用仓库
- name: 从特定仓库安装
  yum:
    name: nginx
    enablerepo: epel
    disablerepo: "*"
    state: present

# 更新所有包
- name: 更新系统
  yum:
    name: '*'
    state: latest
    exclude: kernel*          # 排除内核更新

# 删除包
- name: 卸载
  yum:
    name: httpd
    state: absent
    autoremove: yes
```

---

### 2.3 `pip` - Python 包管理

```yaml
# 安装 Python 包
- name: 安装 requests
  pip:
    name: requests
    state: present
    executable: pip3          # 指定 pip 版本

# 安装多个包
- name: 安装依赖
  pip:
    name:
      - django==4.2
      - psycopg2-binary
      - redis
    state: present

# 从 requirements.txt 安装
- name: 安装项目依赖
  pip:
    requirements: /opt/myapp/requirements.txt
    virtualenv: /opt/myapp/venv  # 指定虚拟环境
    virtualenv_python: python3.11

# 升级包
- name: 升级 pip 本身
  pip:
    name: pip
    state: latest
    executable: pip3

# 使用镜像源
- name: 使用国内源安装
  pip:
    name: flask
    extra_args: "-i https://pypi.tuna.tsinghua.edu.cn/simple"
```

---

### 2.4 `gem` / `npm` / `composer` 等

```yaml
# Ruby Gem
- name: 安装 bundler
  gem:
    name: bundler
    state: present
    user_install: no

# Node.js NPM
- name: 安装全局包
  npm:
    name: pm2
    global: yes
    state: present

# 从 package.json 安装
- name: 安装项目依赖
  npm:
    path: /opt/myapp
    state: present

# PHP Composer
- name: 安装依赖
  composer:
    command: install
    working_dir: /opt/myapp
    no_dev: no                # 是否跳过开发依赖
```

---

## 三、服务管理类模块

### 3.1 `service` - 通用服务管理

```yaml
# 启动服务
- name: 启动 Nginx
  service:
    name: nginx
    state: started            # started|stopped|restarted|reloaded

# 停止服务
- name: 停止 Apache
  service:
    name: httpd
    state: stopped

# 重启服务
- name: 重启服务
  service:
    name: mysql
    state: restarted

# 重载配置（不中断服务）
- name: 重载 Nginx
  service:
    name: nginx
    state: reloaded

# 设置开机自启
- name: 启用服务
  service:
    name: nginx
    enabled: yes              # yes|no

# 同时启动并启用
- name: 启动并启用
  service:
    name: redis
    state: started
    enabled: yes
```

---

### 3.2 `systemd` - Systemd 专用

```yaml
# 基础操作（与 service 类似，但功能更丰富）
- name: 启动服务
  systemd:
    name: nginx
    state: started
    enabled: yes

# 守护进程重载（修改 unit 文件后需要）
- name: 重载 systemd
  systemd:
    daemon_reload: yes

# 启用/禁用服务（创建/删除符号链接）
- name: 启用服务
  systemd:
    name: myapp
    enabled: yes
    masked: no                # 取消屏蔽

# 屏蔽服务（完全禁用）
- name: 屏蔽服务
  systemd:
    name: cups
    masked: yes

# 使用 scope（用户级服务）
- name: 启动用户服务
  systemd:
    name: syncthing
    scope: user
    state: started
```

---

### 3.3 `sysvinit` - 传统 SysV 脚本

```yaml
- name: 管理旧式服务
  sysvinit:
    name: mysql
    state: started
    enabled: yes
    runlevels: 3,5            # 指定运行级别
```

---

## 四、命令执行类模块

### 4.1 `command` - 执行命令（推荐）

```yaml
# 基础执行
- name: 获取系统信息
  command: uname -a
  register: uname_result      # 保存输出到变量

- name: 显示结果
  debug:
    var: uname_result.stdout

# 改变工作目录
- name: 在特定目录执行
  command: ./configure --prefix=/usr/local
  args:
    chdir: /tmp/source        # 先 cd 到此目录
    creates: /tmp/source/Makefile  # 如果文件存在则跳过（幂等性）

# 条件执行
- name: 编译安装
  command: make install
  args:
    chdir: /tmp/source
    creates: /usr/local/bin/app   # 如果已安装则跳过

# 移除文件后执行
- name: 清理后重建
  command: ./build.sh
  args:
    chdir: /opt/app
    removes: /opt/app/build.lock  # 只有此文件存在时才执行
```

**特点**：
- 默认不加载 shell 环境（$HOME、$PATH 等可能不同）
- 不支持管道、重定向、通配符
- 更安全，是默认推荐模块

---

### 4.2 `shell` - Shell 执行

```yaml
# 支持管道和重定向
- name: 查找大文件
  shell: find /var/log -type f -size +100M | head -5
  register: large_files

# 使用管道和过滤
- name: 获取内存使用
  shell: free -m | awk '/^Mem/{print $3}'
  register: mem_used

# 多行命令
- name: 复杂操作
  shell: |
    cd /opt/app
    source venv/bin/activate
    pip install -r requirements.txt
    python manage.py migrate
  args:
    executable: /bin/bash     # 指定 shell

# 使用环境变量
- name: 带环境变量执行
  shell: echo $MY_VAR
  environment:
    MY_VAR: "hello world"

# 忽略错误
- name: 可能失败的命令
  shell: /opt/script.sh
  ignore_errors: yes

# 自定义失败条件
- name: 检查日志
  shell: grep ERROR /var/log/app.log
  register: log_check
  failed_when: log_check.rc == 2  # 只有返回码2才算失败（0=成功，1=无匹配）
  changed_when: false             # 此命令不改变系统状态
```

---

### 4.3 `raw` - 原始 SSH 命令

```yaml
# 不依赖 Python，用于安装 Python 前
- name: 安装 Python（裸机）
  raw: |
    apt-get update
    apt-get install -y python3 python3-apt
  become: yes

# 检查网络连通性
- name: 测试网络
  raw: ping -c 1 8.8.8.8
```

---

### 4.4 `script` - 执行本地脚本

```yaml
# 将本地脚本传输到远程执行
- name: 运行部署脚本
  script: /local/scripts/deploy.sh --env production
  args:
    creates: /opt/app/installed.flag  # 幂等性检查

# 带参数的脚本
- name: 初始化数据库
  script: init_db.sh "{{ db_name }}" "{{ db_user }}"
  args:
    chdir: /tmp
```

---

## 五、用户管理类模块

### 5.1 `user` - 用户管理

```yaml
# 创建用户
- name: 创建应用用户
  user:
    name: appuser
    state: present
    uid: 1001
    group: appgroup           # 主组
    groups: docker,sudo       # 附加组
    shell: /bin/bash
    home: /home/appuser
    create_home: yes          # 创建家目录
    password: "{{ 'password' | password_hash('sha512') }}"  # 加密密码
    comment: "Application User"

# 生成 SSH 密钥对
- name: 创建用户并生成密钥
  user:
    name: deploy
    generate_ssh_key: yes
    ssh_key_type: ed25519
    ssh_key_file: .ssh/id_ed25519

# 删除用户
- name: 删除用户
  user:
    name: olduser
    state: absent
    remove: yes               # 同时删除家目录和邮件池
    force: yes                # 强制删除（即使用户已登录）

# 系统用户
- name: 创建系统用户（无登录）
  user:
    name: nginx
    system: yes
    shell: /usr/sbin/nologin
    create_home: no
```

---

### 5.2 `group` - 组管理

```yaml
# 创建组
- name: 创建应用组
  group:
    name: appgroup
    state: present
    gid: 1001
    system: no

# 删除组
- name: 删除旧组
  group:
    name: oldgroup
    state: absent
```

---

### 5.3 `authorized_key` - SSH 公钥管理

```yaml
# 添加公钥
- name: 添加部署密钥
  authorized_key:
    user: deploy
    state: present
    key: "{{ lookup('file', '/local/keys/deploy.pub') }}"
    comment: "deploy@ansible"
    exclusive: no             # yes=只保留此密钥，删除其他

# 从 GitHub 获取公钥
- name: 添加 GitHub 用户密钥
  authorized_key:
    user: admin
    key: "https://github.com/username.keys"
    state: present

# 管理多个密钥
- name: 批量添加密钥
  authorized_key:
    user: root
    state: present
    key: "{{ item }}"
  loop:
    - "{{ lookup('file', 'keys/alice.pub') }}"
    - "{{ lookup('file', 'keys/bob.pub') }}"
    - "ssh-ed25519 AAAAC3... charlie@example.com"

# 删除密钥
- name: 移除旧密钥
  authorized_key:
    user: deploy
    key: "{{ lookup('file', 'keys/old.pub') }}"
    state: absent
```

---

### 5.4 `known_hosts` - 管理 known_hosts

```yaml
# 添加 GitHub 到 known_hosts
- name: 添加 GitHub
  known_hosts:
    name: github.com
    key: "{{ lookup('pipe', 'ssh-keyscan -t rsa github.com') }}"
    state: present
    path: /home/deploy/.ssh/known_hosts

# 扫描并添加
- name: 扫描服务器密钥
  shell: ssh-keyscan -H {{ item }} >> ~/.ssh/known_hosts
  loop:
    - gitlab.example.com
    - github.com
```

---

## 六、数据库类模块

### 6.1 `mysql_db` - MySQL 数据库

```yaml
# 创建数据库
- name: 创建应用数据库
  mysql_db:
    name: myapp
    state: present
    encoding: utf8mb4
    collation: utf8mb4_unicode_ci
    login_user: root
    login_password: "{{ mysql_root_password }}"
    login_unix_socket: /var/run/mysqld/mysqld.sock  # 本地连接使用 socket

# 删除数据库
- name: 删除测试库
  mysql_db:
    name: test
    state: absent

# 导入 SQL 文件
- name: 导入数据
  mysql_db:
    name: myapp
    state: import
    target: /backup/myapp.sql
    login_user: root
    login_password: "{{ mysql_root_password }}"

# 导出/备份
- name: 备份数据库
  mysql_db:
    name: myapp
    state: dump
    target: "/backup/myapp_{{ ansible_date_time.date }}.sql.gz"
    login_user: backup
    login_password: "{{ backup_password }}"
```

---

### 6.2 `mysql_user` - MySQL 用户

```yaml
# 创建用户并授权
- name: 创建应用用户
  mysql_user:
    name: appuser
    password: "{{ app_db_password }}"
    priv: "myapp.*:ALL"       # 数据库.表:权限
    host: '%'                 # 允许从任何主机连接
    state: present
    login_user: root
    login_password: "{{ mysql_root_password }}"

# 精细化授权
- name: 只读用户
  mysql_user:
    name: readonly
    password: "{{ ro_password }}"
    priv:
      'myapp.*': 'SELECT,SHOW VIEW'  # 仅查询权限
      'reporting.*': 'SELECT'
    host: '192.168.1.%'
    state: present

# 撤销权限
- name: 修改权限
  mysql_user:
    name: olduser
    priv: "myapp.*:SELECT"
    append_privs: no          # no=覆盖，yes=追加
    state: present

# 删除用户
- name: 删除用户
  mysql_user:
    name: tempuser
    state: absent
    host: 'localhost'
```

---

### 6.3 `postgresql_db` / `postgresql_user`

```yaml
# PostgreSQL 类似用法
- name: 创建数据库
  postgresql_db:
    name: myapp
    state: present
    encoding: UTF-8
    lc_collate: en_US.UTF-8
    lc_ctype: en_US.UTF-8
    owner: appuser

- name: 创建用户
  postgresql_user:
    name: appuser
    password: "{{ pg_password }}"
    db: myapp
    priv: ALL
    state: present
```

---

## 七、网络类模块

### 7.1 `uri` - HTTP 请求

```yaml
# GET 请求
- name: 检查 API 健康
  uri:
    url: http://localhost:8080/health
    method: GET
    status_code: 200
    return_content: yes
  register: health_check

- name: 显示响应
  debug:
    var: health_check.json

# POST 请求（JSON）
- name: 创建用户
  uri:
    url: https://api.example.com/users
    method: POST
    body:
      name: "John Doe"
      email: "john@example.com"
    body_format: json
    headers:
      Authorization: "Bearer {{ api_token }}"
      Content-Type: "application/json"
    status_code: 201
  register: create_result

# POST 表单数据
- name: 提交表单
  uri:
    url: https://example.com/login
    method: POST
    body: "username=admin&password={{ admin_pass }}"
    headers:
      Content-Type: "application/x-www-form-urlencoded"
    status_code: 302

# 下载文件
- name: 下载安装包
  uri:
    url: https://example.com/app.tar.gz
    method: GET
    dest: /tmp/app.tar.gz
    mode: '0644'
```

---

### 7.2 `get_url` - 下载文件

```yaml
# 基础下载
- name: 下载 JDK
  get_url:
    url: https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.tar.gz
    dest: /opt/jdk-17.tar.gz
    mode: '0644'
    timeout: 300

# 验证校验和
- name: 安全下载
  get_url:
    url: https://example.com/app.tar.gz
    dest: /opt/app.tar.gz
    checksum: sha256:abc123...def456  # 或 sha256:https://example.com/app.tar.gz.sha256

# 使用代理
- name: 通过代理下载
  get_url:
    url: https://external.com/file.zip
    dest: /tmp/file.zip
    use_proxy: yes
    url_username: proxyuser
    url_password: proxypass

# 强制重新下载
- name: 更新文件
  get_url:
    url: https://example.com/config.json
    dest: /etc/app/config.json
    force: yes                # 即使文件存在也重新下载
```

---

### 7.3 `wait_for` - 等待条件

```yaml
# 等待端口可用
- name: 等待服务启动
  wait_for:
    port: 8080
    host: 127.0.0.1
    delay: 5                  # 延迟5秒开始检查
    timeout: 300              # 最多等待300秒
    state: started            # started|stopped

# 等待文件存在
- name: 等待日志文件生成
  wait_for:
    path: /var/log/app/startup.log
    state: present
    delay: 10

# 等待文件包含特定内容
- name: 等待启动完成
  wait_for:
    path: /var/log/app/app.log
    search_regex: "Server started"
    timeout: 120

# 等待连接断开（服务停止）
- name: 等待服务停止
  wait_for:
    port: 8080
    state: stopped
    timeout: 60

# 等待自定义条件
- name: 等待数据库就绪
  shell: mysqladmin ping
  register: result
  until: result.rc == 0
  retries: 10
  delay: 5
```

---

## 八、云平台和容器类

### 8.1 `docker_container` - Docker 容器

```yaml
# 运行容器
- name: 启动 Nginx 容器
  docker_container:
    name: web
    image: nginx:alpine
    state: started
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /local/nginx.conf:/etc/nginx/nginx.conf:ro
      - /data/www:/usr/share/nginx/html:ro
    env:
      NGINX_HOST: example.com
    restart_policy: always
    memory: 512m
    cpus: '1.0'

# 停止并删除
- name: 停止容器
  docker_container:
    name: web
    state: absent               # stopped|started|absent
    force_kill: yes

# 批量管理
- name: 启动多个服务
  docker_container:
    name: "{{ item.name }}"
    image: "{{ item.image }}"
    ports: "{{ item.ports }}"
    state: started
  loop:
    - { name: 'db', image: 'mysql:8.0', ports: ['3306:3306'] }
    - { name: 'cache', image: 'redis:alpine', ports: ['6379:6379'] }
```

---

### 8.2 `docker_image` - Docker 镜像

```yaml
# 拉取镜像
- name: 拉取最新镜像
  docker_image:
    name: nginx
    tag: alpine
    source: pull

# 构建镜像
- name: 构建应用镜像
  docker_image:
    name: myapp
    tag: "{{ app_version }}"
    build:
      path: /opt/myapp          # Dockerfile 所在目录
      dockerfile: Dockerfile.prod
      args:
        BUILD_ENV: production
    source: build
    push: yes                   # 构建后推送
    repository: registry.example.com/myapp

# 删除镜像
- name: 清理旧镜像
  docker_image:
    name: myapp
    tag: old
    state: absent
```

---

### 8.3 `k8s` - Kubernetes

```yaml
# 部署应用
- name: 创建 Deployment
  k8s:
    state: present
    definition:
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: web
        namespace: default
      spec:
        replicas: 3
        selector:
          matchLabels:
            app: web
        template:
          metadata:
            labels:
              app: web
          spec:
            containers:
              - name: nginx
                image: nginx:alpine
                ports:
                  - containerPort: 80

# 使用模板文件
- name: 从文件部署
  k8s:
    state: present
    src: /local/k8s/app.yaml
    namespace: production
```

---

## 九、其他实用模块

### 9.1 `cron` - 定时任务

```yaml
# 创建定时任务
- name: 添加备份任务
  cron:
    name: "Database backup"
    minute: "0"
    hour: "2"
    day: "*"
    month: "*"
    weekday: "*"
    job: "/opt/scripts/backup.sh"
    user: backup
    state: present

# 特殊时间
- name: 每周日执行
  cron:
    name: "Weekly cleanup"
    special_time: weekly      # reboot|yearly|annually|monthly|weekly|daily|hourly
    job: "/opt/scripts/cleanup.sh"

# 删除任务
- name: 移除旧任务
  cron:
    name: "Old job"
    state: absent

# 禁用任务（注释掉）
- name: 暂停任务
  cron:
    name: "Maintenance"
    job: "/opt/maintenance.sh"
    disabled: yes
```

---

### 9.2 `mount` - 挂载管理

```yaml
# 挂载 NFS
- name: 挂载共享存储
  mount:
    path: /data/shared
    src: nfs.example.com:/exports/data
    fstype: nfs
    opts: rw,sync,no_subtree_check
    state: mounted            # mounted|unmounted|absent|present

# 卸载
- name: 卸载
  mount:
    path: /data/shared
    state: unmounted

# 从 fstab 删除
- name: 清理 fstab
  mount:
    path: /data/old
    state: absent
```

---

### 9.3 `debug` - 调试输出

```yaml
# 打印变量
- name: 显示变量
  debug:
    var: ansible_hostname

# 自定义消息
- name: 显示信息
  debug:
    msg: "Deploying to {{ inventory_hostname }} ({{ ansible_default_ipv4.address }})"

# 条件输出
- name: 警告
  debug:
    msg: "磁盘空间不足！"
  when: ansible_mounts[0].size_available < 1073741824  # 1GB

# 美化输出
- name: 显示 JSON
  debug:
    var: complex_var
    verbosity: 2              # 只有 -vv 以上才显示
```

---

### 9.4 `set_fact` - 设置变量

```yaml
# 动态设置变量
- name: 计算版本号
  set_fact:
    full_version: "{{ app_version }}-{{ build_number }}"
    deploy_time: "{{ ansible_date_time.iso8601 }}"

# 合并字典
- name: 合并配置
  set_fact:
    final_config: "{{ default_config | combine(user_config, recursive=true) }}"

# 基于条件设置
- name: 设置环境
  set_fact:
    env_config: "{{ prod_config }}"
  when: deployment_environment == "production"
```

---

### 9.5 `include_tasks` / `import_tasks` - 任务复用

```yaml
# 动态包含（运行时处理）
- name: 包含特定系统任务
  include_tasks: "{{ ansible_os_family }}.yml"

# 静态导入（解析时处理）
- name: 导入通用任务
  import_tasks: common.yml
  tags: ['common']

# 带参数的包含
- name: 部署应用
  include_tasks: deploy_app.yml
  vars:
    app_name: frontend
    app_port: 3000
```

---

## 十、模块使用最佳实践

### 10.1 幂等性原则

```yaml
# ✅ 好的做法（幂等）
- name: 确保服务运行
  service:
    name: nginx
    state: started
    enabled: yes

# ❌ 避免的做法（非幂等）
- name: 启动服务
  command: systemctl start nginx
```

### 10.2 错误处理

```yaml
- name: 尝试操作
  block:
    - name: 可能失败的任务
      command: /risky/script.sh
  rescue:
    - name: 回滚操作
      command: /rollback/script.sh
  always:
    - name: 清理
      file:
        path: /tmp/temp
        state: absent
```

### 10.3 条件判断

```yaml
- name: 仅对特定系统执行
  apt:
    name: nginx
    state: present
  when: 
    - ansible_os_family == "Debian"
    - ansible_distribution_version is version('20.04', '>=')
```

### 10.4 循环优化

```yaml
# 批量操作（比循环更高效）
- name: 安装多个包
  apt:
    name:
      - vim
      - git
      - curl
    state: present

# 需要不同参数时用循环
- name: 创建多个用户
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: 'alice', groups: 'sudo' }
    - { name: 'bob', groups: 'developers' }
```

---

这些模块覆盖了文件操作、包管理、服务控制、用户管理、数据库、网络、容器等各个方面。建议通过 `ansible-doc 模块名` 查看每个模块的详细文档，并在实际使用中结合 `check` 模式进行测试。