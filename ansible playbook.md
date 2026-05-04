 
# Playbook 怎么写

### 1.1 基本结构
* 最小结构组成
```yaml
---
- hosts: all          # 唯一必须：目标主机
  tasks:              # 通常需要：要执行的任务
    - name: Hello
      debug:
        msg: "Hello World"
```
* 完整结构参考
```yaml
---
# ═══════════════════════════════════════
# PLAY 1: 全局配置
# ═══════════════════════════════════════

- name: 部署 Web 应用              # 可选：描述性名称，建议有
  hosts: webservers               # ███ 必须：目标主机或组
  gather_facts: yes               # 可选：默认 yes，收集主机信息
  become: yes                     # 可选：是否提权（sudo）
  become_user: root               # 可选：提权到哪个用户
  
  # ─── 变量定义（全部可选）───
  vars:                           # 直接定义变量
    app_version: "1.2.3"
    deploy_path: "/var/www"
  
  vars_files:                     # 从文件加载变量
    - vars/common.yml
    - "vars/{{ ansible_os_family }}.yml"   # 动态文件名
  
  vars_prompt:                    # 交互式输入变量（运行时提示）
    - name: db_password
      prompt: "Enter DB password"
      private: yes                # 密码不显示
  
  # ─── 角色调用（可选）───
  roles:                          # 按顺序执行 roles
    - common
    - { role: nginx, nginx_port: 8080 }   # 带参数
    - { role: app, when: "env == 'prod'" } # 条件执行
  
  # ─── 任务执行顺序 ───
  pre_tasks:                      # 可选：role 之前执行
    - name: 更新 apt 缓存
      apt:
        update_cache: yes
  
  tasks:                          # ███ 核心：主要任务列表
    - name: 复制应用代码
      copy:
        src: ./app/
        dest: "{{ deploy_path }}"
      notify: restart app         # 触发 handler
  
  post_tasks:                     # 可选：role 之后执行
    - name: 健康检查
      uri:
        url: "http://localhost/health"
        status_code: 200
  
  # ─── 触发器（可选）───
  handlers:                       # 被 notify 触发
    - name: restart app
      service:
        name: myapp
        state: restarted
  
  # ─── 执行控制（可选）───
  serial: 2                       # 分批执行，每次2台
  max_fail_percentage: 30         # 失败超过30%停止
  any_errors_fatal: yes             # 任何错误立即中止

# ═══════════════════════════════════════
# PLAY 2: 数据库配置（另一个 Play）
# ═══════════════════════════════════════
- name: 配置数据库
  hosts: dbservers
  roles:
    - mysql
```
### 1.3 关键语法详解

#### 1.3.1 变量使用

```yaml
# 定义变量
vars:
  app_name: myapp
  version: "2.0"

# 使用变量（Jinja2 语法）
tasks:
  - name: 创建应用目录
    file:
      path: "/opt/{{ app_name }}-{{ version }}"
      state: directory
  
  # 获取 facts 变量（自动收集的系统信息）
  - name: 显示操作系统
    debug:
      msg: "当前系统: {{ ansible_distribution }} {{ ansible_distribution_version }}"
  
  # 注册变量（保存任务输出）
  - name: 检查磁盘空间
    command: df -h /
    register: disk_result
  
  - name: 显示磁盘信息
    debug:
      var: disk_result.stdout_lines
```

#### 1.3.2 条件判断

```yaml
tasks:
  # 基于操作系统
  - name: 安装 Nginx (CentOS)
    yum:
      name: nginx
      state: present
    when: ansible_os_family == "RedHat"
  
  - name: 安装 Nginx (Ubuntu)
    apt:
      name: nginx
      state: present
    when: ansible_os_family == "Debian"
  
  # 基于变量值
  - name: 生产环境配置
    template:
      src: prod.conf.j2
      dest: /etc/app/config.conf
    when: env == "production"
  
  # 多条件组合
  - name: 复杂条件
    debug:
      msg: "满足条件"
    when: 
      - ansible_distribution == "Ubuntu"
      - ansible_distribution_version is version('20.04', '>=')
      - env is defined
```

#### 1.3.3 循环

```yaml
tasks:
  # 简单列表循环
  - name: 安装多个包
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - vim
      - git
      - curl
      - htop
  
  # 哈希循环
  - name: 创建用户
    user:
      name: "{{ item.name }}"
      state: present
      groups: "{{ item.groups }}"
    loop:
      - { name: 'alice', groups: 'sudo' }
      - { name: 'bob', groups: 'developers' }
      - { name: 'charlie', groups: 'sudo,developers' }
  
  # 循环字典
  - name: 配置环境变量
    lineinfile:
      path: /etc/environment
      line: "{{ item.key }}={{ item.value }}"
    loop: "{{ env_vars | dict2items }}"
    vars:
      env_vars:
        JAVA_HOME: /usr/lib/jvm/java-11
        MAVEN_HOME: /opt/maven
  
  # 使用 with_items（旧语法，仍兼容）
  - name: 旧语法循环
    debug:
      msg: "{{ item }}"
    with_items:
      - one
      - two
```

#### 1.3.4 错误处理

```yaml
tasks:
  # 忽略错误继续执行
  - name: 尝试停止可能不存在的服务
    service:
      name: old-service
      state: stopped
    ignore_errors: yes
  
  # 自定义错误条件
  - name: 运行脚本
    command: /opt/script.sh
    register: script_result
    failed_when: "'ERROR' in script_result.stderr"
  
  # 变更检测
  - name: 检查配置文件
    command: test -f /etc/app/config.conf
    register: config_check
    changed_when: false  # 此命令不改变系统状态
```

#### 1.3.5 块与异常处理

```yaml
tasks:
  - name: 数据库操作块
    block:
      - name: 备份数据库
        mysql_db:
          state: dump
          name: mydb
          target: /backup/mydb.sql
      
      - name: 执行迁移脚本
        command: /opt/migrate.sh
    
    rescue:
      - name: 迁移失败，恢复备份
        mysql_db:
          state: import
          name: mydb
          target: /backup/mydb.sql
      
      - name: 发送告警
        mail:
          to: admin@example.com
          subject: "迁移失败"
          body: "数据库迁移失败，已回滚"
    
    always:
      - name: 清理临时文件
        file:
          path: /tmp/migration.tmp
          state: absent
```

---
