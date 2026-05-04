## Ansible 命令行常用命令

### 2.1 命令速查表

| 命令                  | 用途           | 常用场景            |
| ------------------- | ------------ | --------------- |
| `ansible`           | 执行 ad-hoc 命令 | 快速检查、临时操作       |
| `ansible-playbook`  | 运行剧本         | 正式部署、自动化任务      |
| `ansible-doc`       | 查看模块文档       | 学习模块用法          |
| `ansible-inventory` | 管理主机清单       | 查看、验证 Inventory |
| `ansible-galaxy`    | 管理角色和集合      | 下载社区角色          |
| `ansible-vault`     | 加密敏感数据       | 保护密码、密钥         |
| `ansible-config`    | 查看配置         | 检查 Ansible 配置   |
| `ansible-pull`      | 拉取模式执行       | 客户端主动执行         |

---

### 2.2 详细用法与示例

#### 2.2.1 `ansible` - Ad-hoc 命令

用于快速执行单次任务，无需编写 Playbook。

```bash
# ========== 基础连通性测试 ==========
ansible all -m ping
ansible webservers -m ping

# ========== 执行命令 ==========
# 在所有主机执行 uptime
ansible all -m command -a "uptime"

# 使用 shell 模块（支持管道、重定向）
ansible webservers -m shell -a "df -h | grep /dev/sda"

# 使用 raw 模块（不依赖 Python，用于安装 Python 前）
ansible all -m raw -a "apt-get update"

# ========== 文件操作 ==========
# 复制文件
ansible webservers -m copy -a "src=/local/file.txt dest=/remote/file.txt mode=644"

# 创建目录
ansible all -m file -a "path=/opt/myapp state=directory mode=755 owner=root"

# 删除文件
ansible all -m file -a "path=/tmp/oldfile state=absent"

# ========== 包管理 ==========
# Ubuntu/Debian
ansible webservers -m apt -a "name=nginx state=present" -b

# 更新缓存并安装
ansible webservers -m apt -a "name=vim state=present update_cache=yes" -b

# CentOS/RHEL
ansible webservers -m yum -a "name=httpd state=present" -b

# ========== 服务管理 ==========
ansible webservers -m service -a "name=nginx state=started enabled=yes" -b

# 重启服务
ansible webservers -m service -a "name=nginx state=restarted" -b

# ========== 用户管理 ==========
ansible all -m user -a "name=deploy state=present groups=sudo shell=/bin/bash" -b

# ========== 收集 Facts ==========
ansible webservers -m setup
ansible webservers -m setup -a "filter=ansible_memory_mb"
```

**常用参数说明：**

| 参数 | 说明 |
|------|------|
| `-m` | 指定模块 |
| `-a` | 模块参数 |
| `-b` / `--become` | 提权（sudo） |
| `-K` / `--ask-become-pass` | 询问 sudo 密码 |
| `-k` / `--ask-pass` | 询问 SSH 密码 |
| `-i` | 指定 inventory 文件 |
| `-u` | 指定远程用户 |
| `-C` / `--check` | 干运行（不实际执行） |
| `-D` / `--diff` | 显示变更差异 |
| `-v` / `-vvv` | 详细输出（最多3个v） |
| `-l` / `--limit` | 限制特定主机 |
| `-t` / `--tags` | 执行指定标签的任务 |
| `--skip-tags` | 跳过指定标签 |

---

#### 2.2.2 `ansible-playbook` - 运行剧本

```bash
# ========== 基础执行 ==========
ansible-playbook site.yml

# 指定 inventory
ansible-playbook -i production.ini site.yml

# 指定用户和密码
ansible-playbook -i hosts site.yml -u admin -k -K

# 使用 SSH 密钥
ansible-playbook site.yml --private-key ~/.ssh/id_rsa

# ========== 调试与测试 ==========
# 语法检查
ansible-playbook site.yml --syntax-check

# 干运行（预览变更，不实际执行）
ansible-playbook site.yml --check

# 显示详细差异
ansible-playbook site.yml --check --diff

# 详细输出（调试）
ansible-playbook site.yml -vvv

# 步进执行（每步确认）
ansible-playbook site.yml --step

# 从特定任务开始
ansible-playbook site.yml --start-at-task="安装 Nginx"

# ========== 选择性执行 ==========
# 仅执行特定标签
ansible-playbook site.yml --tags "install,config"

# 跳过特定标签
ansible-playbook site.yml --skip-tags "test"

# 限制特定主机/组
ansible-playbook site.yml --limit webservers
ansible-playbook site.yml --limit "web1.example.com"

# 使用模式匹配
ansible-playbook site.yml --limit "~web.*"

# ========== 变量传递 ==========
# 命令行传递变量
ansible-playbook site.yml -e "env=production version=2.0"

# 传递多个变量
ansible-playbook site.yml -e '{"env":"prod","debug":false}'

# 从文件加载变量
ansible-playbook site.yml -e "@vars.json"
ansible-playbook site.yml -e "@vars.yml"

# ========== 并发控制 ==========
# 调整并发数（默认5）
ansible-playbook site.yml -f 20

# 串行执行（一批一台）
ansible-playbook site.yml --forks 1

# Playbook 内设置串行
# serial: 1  # 每次一台
# serial: "30%"  # 每次30%的主机

# ========== 其他实用选项 ==========
# 列出主机（不执行）
ansible-playbook site.yml --list-hosts

# 列出任务
ansible-playbook site.yml --list-tasks

# 列出标签
ansible-playbook site.yml --list-tags

# 超时设置
ansible-playbook site.yml --timeout 30
```

---

#### 2.2.3 `ansible-doc` - 模块文档

```bash
# 列出所有模块
ansible-doc -l

# 查看特定模块文档
ansible-doc apt
ansible-doc service
ansible-doc copy

# 查看精简版帮助
ansible-doc -s apt

# 搜索模块
ansible-doc -l | grep mysql

# 查看插件文档
ansible-doc -t lookup file
```

---

#### 2.2.4 `ansible-inventory` - 清单管理

```bash
# 显示 inventory 信息
ansible-inventory --list

# 以 YAML 格式显示
ansible-inventory --list --yaml

# 以图形式显示
ansible-inventory --graph

# 查看特定主机信息
ansible-inventory --host web1.example.com

# 验证 inventory 文件
ansible-inventory -i custom_hosts.ini --list

# 查看组内主机
ansible-inventory --graph webservers
```

---

#### 2.2.5 `ansible-galaxy` - 角色管理

```bash
# ========== 角色操作 ==========
# 搜索角色
ansible-galaxy search nginx

# 安装角色
ansible-galaxy install geerlingguy.nginx

# 指定版本安装
ansible-galaxy install geerlingguy.nginx,3.1.0

# 从 requirements 文件安装
ansible-galaxy install -r requirements.yml

# 示例 requirements.yml：
# ---
# roles:
#   - name: geerlingguy.nginx
#     version: 3.1.0
#   - src: https://github.com/custom/role.git
#     scm: git
#     version: main

# 列出已安装角色
ansible-galaxy list

# 删除角色
ansible-galaxy remove geerlingguy.nginx

# 初始化新角色（创建目录结构）
ansible-galaxy init myrole

# ========== 集合操作（Ansible 2.9+） ==========
# 安装集合
ansible-galaxy collection install community.mysql

# 从 galaxy.yml 安装
ansible-galaxy collection install -r requirements.yml
```

---

#### 2.2.6 `ansible-vault` - 加密管理

```bash
# 创建加密文件
ansible-vault create secrets.yml

# 编辑加密文件
ansible-vault edit secrets.yml

# 查看加密文件
ansible-vault view secrets.yml

# 加密现有文件
ansible-vault encrypt secrets.yml

# 解密文件
ansible-vault decrypt secrets.yml

# 更改密码
ansible-vault rekey secrets.yml

# 加密字符串（用于嵌入 Playbook）
ansible-vault encrypt_string 'my_secret_password' --name 'db_password'

# 使用加密文件运行 Playbook
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file .vault_pass
ansible-playbook site.yml --vault-password-file /path/to/script.sh

# 使用密码文件（自动化场景）
echo "mypassword" > .vault_pass
chmod 600 .vault_pass
ansible-playbook site.yml --vault-password-file .vault_pass
```

---

#### 2.2.7 `ansible-config` - 配置查看

```bash
# 查看当前配置
ansible-config view

# 查看所有配置（包括默认值）
ansible-config dump

# 查看特定配置项
ansible-config dump | grep DEFAULT_FORKS

# 列出所有配置选项
ansible-config list
```

---

### 2.3 实际场景命令组合

```bash
# ========== 场景1：快速巡检 ==========
ansible all -m shell -a "uptime && free -h && df -h" | tee server_status.txt

# ========== 场景2：批量部署公钥 ==========
ansible all -m authorized_key -a "user=root key='{{ lookup('file', '~/.ssh/id_rsa.pub') }}'" -k

# ========== 场景3：滚动更新（每次2台） ==========
ansible-playbook deploy.yml --limit webservers -f 2

# ========== 场景4：紧急回滚 ==========
ansible-playbook rollback.yml --tags "emergency" -e "version=1.9"

# ========== 场景5：安全加固检查 ==========
ansible-playbook security.yml --check --diff -vvv | grep -E "(changed|failed)"

# ========== 场景6：定时任务批量添加 ==========
ansible all -m cron -a "name='backup' minute='0' hour='2' job='/opt/backup.sh'"

# ========== 场景7：收集所有服务器信息 ==========
ansible all -m setup --tree /tmp/facts
```

---

### 2.4 调试技巧

```bash
# 最高级别调试
ANSIBLE_DEBUG=1 ansible-playbook site.yml -vvvvv

# 显示任务耗时
ansible-playbook site.yml --profile-tasks

# 回调插件显示时间
export CALLBACKS_ENABLED=profile_tasks

# 检查特定主机的变量
ansible web1.example.com -m debug -a "var=hostvars[inventory_hostname]"
```

掌握这些命令和 Playbook 写法，可以覆盖 90% 的 Ansible 使用场景。建议从简单的 ad-hoc 命令开始，逐步过渡到复杂的 Playbook 和 Roles 架构。