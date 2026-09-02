# Ansible 运维语法实战笔记

> 基于 maxs-ops 工程中 220+ playbook、196 个 role、200+ 模板的模式整理

---

## 一、ansible.cfg 核心配置

```ini
[defaults]
inventory = inventory/hosts              # 主机清单路径
roles_path = /data/maxs-ops/roles        # 角色搜索路径
host_key_checking = False                # 跳过 SSH 密钥验证
gathering = smart                         # 智能事实采集
gather_timeout = 7                        # 采集超时
remote_tmp = /data/.ansible/tmp          # 远程临时目录
log_path = /var/log/ansible.log          # 日志文件
private_role_vars = True                  # 角色变量隔离
error_on_undefined_vars = True            # 未定义变量报错
retry_files_enabled = False               # 禁用 retry 文件
display_skipped_hosts = False             # 隐藏跳过的 host
stdout_callback = task_timer              # 自定义输出回调
callback_plugins = ./callback_plugins     # 回调插件目录

[ssh_connection]
ssh_args = -C -o ControlMaster=auto -o ControlPersist=10m
pipelining = True                         # 减少 SSH 连接（性能优化）
control_path = /tmp/ansible-ssh-%%h-%%p-%%r
transfer_method = scp
retries = 10

# 事实缓存（加速后续执行）
fact_caching = jsonfile
fact_caching_connection = ./.tmp
fact_caching_timeout = 3600
```

---

## 二、Inventory 主机清单（INI 格式）

### 2.1 基本结构

```ini
# 单机
[oper_master]
10.21.18.151 ansible_connection=local

# 主机组（带变量）
[k8s_master]
10.21.18.151 lables=role:maxs,role2:flink

[nginx]
10.21.18.151 NGINX_ROLE=MASTER

[es]
10.21.18.151 NODE_ROLES="master"

# 组嵌套
[flink:children]
flink_master

[flink:vars]
version=1.16.0
mode=K8S
```

### 2.2 命名规范

```ini
# 组名：[a-z0-9]([-a-z0-9]*[a-z0-9])?
# IP + 行内变量用空格分隔
10.21.18.151 ansible_connection=local TYPE=master
```

---

## 三、Playbook 结构

### 3.1 最小 playbook

```yaml
- hosts: es
  roles:
    - es
  gather_facts: False
```

### 3.2 带条件的 role

```yaml
- hosts:
    - all_server
  roles:
    - { role: chrony, when: "groups['all_server']|length > 1" }
```

### 3.3 任务型 playbook（start.yml 实例）

```yaml
- hosts: mysql
  tags: mysql
  tasks:
    - name: starting mysql cluster
      service: name=asap_mysql state=started

- hosts: flink
  tags: flink
  tasks:
    - name: starting flink job cluster
      service:
        name: asap_flink_jobmanager
        state: started
      when:
        - "groups['flink']|length > 1"
        - "inventory_hostname in groups['flink_master']"
        - "mode is undefined or mode != 'yarn'"
```

**注意**：`tags:` 放在 play 级别（与 `hosts:` 同级）会影响整个 play 下的所有 task。

### 3.4 变量型 playbook（通过 extra-vars 传参）

```yaml
- hosts: "{{ NODE_TO_ADD }}"
  tasks:
    - name: 获取服务信息
      service_facts:

    - name: 已安装则重启
      import_role:
        name: es
      when: "ansible_facts['services']['asap_es.service'] is not defined"

    - name: 调用 role
      include_role:
        name: es
      when: "ansible_facts['services']['asap_es.service'] is not defined"
```

---

## 四、核心模块速查

### 4.1 shell —— 执行 Shell 命令

```yaml
- name: "单行命令"
  shell: systemctl daemon-reload && systemctl restart asap_es

- name: "多行命令"
  shell: |
    cd /data/maxs-ops
    ansible-playbook -t add_es playbooks/add/add-es-tag.yml

- name: "注册结果 + 轮询"
  shell: "systemctl is-active asap_es.service"
  register: svc_status
  until: '"active" in svc_status.stdout'
  retries: 20
  delay: 10
```

### 4.2 service —— 管理 systemd 服务

```yaml
- name: starting mysql
  service: name=asap_mysql state=started     # 简写

- name: starting etcd
  service:                                    # 完整写法
    name: asap_etcd
    state: started

# state 可选值: started / stopped / restarted / reloaded
```

### 4.3 template —— 分发 Jinja2 模板

```yaml
- name: 分发配置文件
  template:
    src: "server.properties.j2"
    dest: "{{ kafka_config_dir }}/server.properties"
    owner: asap
    group: asap
    mode: 0755

- name: 批量分发模板
  template:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    owner: asap
    group: asap
    mode: 0755
  with_items:
    - { src: 'producer.properties.j2', dest: '{{ kafka_config_dir }}/producer.properties' }
    - { src: 'consumer.properties.j2', dest: '{{ kafka_config_dir }}/consumer.properties' }
    - { src: 'asap_kafka.service.j2', dest: '/etc/systemd/system/asap_kafka.service' }
```

### 4.4 copy —— 复制文件（非模板）

```yaml
- name: 分发 SSL 文件
  copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    owner: asap
    group: asap
    mode: 0755
  with_items:
    - { src: 'server.truststore.jks', dest: '{{ kafka_config_dir }}/server.truststore.jks' }
    - { src: 'server.keystore.jks', dest: '{{ kafka_config_dir }}/server.keystore.jks' }
```

### 4.5 stat —— 检查文件/路径状态

```yaml
- name: "判断 es 是否已安装"
  stat:
    path: "{{ es_dir }}"
  register: srv_status
# 之后用 srv_status.stat.exists 判断是否存在
```

### 4.6 service_facts —— 获取服务状态

```yaml
- name: 获取服务信息
  service_facts:
# 之后用 ansible_facts['services']['asap_es.service'] 判断服务是否存在
```

### 4.7 debug —— 打印调试信息

```yaml
- name: "打印集群状态"
  debug:
    var: es_cluster_status.stdout_lines

- name: "打印变量值"
  debug:
    msg: "broker_id 的值是 {{ broker_id }}"
```

### 4.8 set_fact —— 动态设置变量

```yaml
- set_fact:
    broker_id: "{{ broker_id.stdout }}"

- set_fact:
    index_exists: "{{ index_check.rc == 0 and index_check.stdout != '[]' }}"
```

### 4.9 file —— 文件/目录操作

```yaml
- name: "创建目录"
  file:
    name: "{{ install_dir }}"
    state: directory
    owner: "{{ user }}"
    group: "{{ group }}"
```

### 4.10 unarchive —— 解压

```yaml
- name: "解压压缩包"
  unarchive:
    src: "{{ tar_dir }}"
    dest: "{{ install_dir }}"
    owner: "{{ user }}"
    group: "{{ group }}"
```

---

## 五、执行控制选项

### 5.1 when —— 条件执行

```yaml
# 简单条件
when: "groups['etcd']|length > 1"

# 多个条件（AND）
when:
  - "groups['flink']|length > 1"
  - "mode is undefined or mode != 'yarn'"

# 使用变量
when: 'CLICKHOUSE_OPEN_FLAG == "1"'

# 变量 + 默认值 + 数字比较
when: "(DEL_FLAG | default(0)) != 1"

# Ansible facts
when: 'ansible_distribution in ["Kylin Linux Advanced Server"] and ansible_architecture != "aarch64"'

# 在特定主机上执行
when: "inventory_hostname == groups.kafka[0]"

# 检查变量是否定义
when: 'PARAM_KAFKA_AUTH is defined and PARAM_KAFKA_AUTH == "KERBEROS"'
```

### 5.2 register + until（轮询等待）

```yaml
- name: "轮询启动服务，并检查状态"
  shell: "systemctl is-active asap_es.service"
  register: svc_status
  until: '"active" in svc_status.stdout'
  retries: 10
  delay: 3
```

### 5.3 delegate_to —— 委托到其他主机执行

```yaml
- name: 在 oper_master 上执行
  shell: |
    cd /data/maxs-ops
    ansible-playbook -t es playbooks/system/enable.yml
  delegate_to: "{{ groups['oper_master'][0] }}"
```

### 5.4 run_once —— 只执行一次

```yaml
- name: start vastbase cluster
  shell: |
    source /home/vastbase/.Vastbase
    cm_ctl start
  become: yes
  become_user: vastbase
  run_once: true
  delegate_to: "{{ groups['vastbase'][0] }}"
```

### 5.5 connection: local —— 在本机执行

```yaml
- name: "生成映射文件"
  shell: "sh /data/maxs-ops/tools/create_master_mapping.sh kafka"
  connection: local
```

### 5.6 block —— 任务分组 + 条件

```yaml
- block:
    - name: "删除索引"
      shell: curl -XDELETE "https://{{ host }}:9200/{{ index }}"
      register: delete_result

    - name: "打印结果"
      debug:
        msg: "{{ delete_result.stdout }}"
  when: index_exists
  tags:
    - cleanup_index
```

### 5.7 ignore_errors —— 忽略错误

```yaml
- name: "强制安装 rpm"
  shell: rpm -Uvh {{ rpm_dir }}/kerberos/client/*.rpm --force --nodeps
  ignore_errors: true
```

### 5.8 import_role vs include_role

```yaml
# import_role：静态加载（编译时决定，条件必须用简单变量）
- import_role:
    name: es
  when: "ansible_facts['services']['asap_es.service'] is not defined"

# include_role：动态加载（运行时决定，支持复杂条件）
- include_role:
    name: elasticsearch_exporter
  when: "ansible_facts['services']['asap_es.service'] is not defined"
```

---

## 六、Ansible 命令行

### 6.1 ansible-playbook

```bash
# 基本执行
ansible-playbook playbooks/es/es.yml

# 限制目标主机
ansible-playbook -l "10.21.18.151" playbooks/es/es.yml
ansible-playbook --limit "$host" playbooks/business/11.file-scp-exec.yml

# 按标签执行
ansible-playbook -t add_es playbooks/add/add-es-tag.yml
ansible-playbook -t refresh_sysctl_config playbooks/k8s/01.prepare.yml

# 传递变量
ansible-playbook --extra-vars "ip=$name keys=$item" playbooks/k8s/17.k8s-add-lable.yml
ansible-playbook -e "HOST=${value1} SERVICE_NAME=${value3} OPER=started" playbooks/system/template.yml
ansible-playbook --extra-vars "collect_all=$collect_all module_type=$module_type" playbooks/common/41.log-collection.yml

# 跳过标签
ansible-playbook /data/maxs-ops/playbooks/precheck/base.yaml --skip-tags=port,uid,time

# 并发度控制
ansible-playbook playbooks/destroy/25.files_and_vars.yml --forks 2
```

### 6.2 ansible ad-hoc

```bash
# 在所有 server 上执行命令
ansible all_server -m command -a 'date'

# 获取特定信息
ansible $pwdm_ip -m debug -a "msg={{hostvars[inventory_hostname]['hostname']}}"

# 列出组内主机
ansible -i /data/maxs-ops/inventory/hosts oper_master --list-hosts
```

---

## 七、Jinja2 模板语法（在 .j2 文件中）

### 7.1 变量输出

```jinja2
broker.id={{ broker_id }}
cluster.name: {{ cluster_name }}
network.host: {{ hostvars[inventory_hostname]['hostname'] }}
discovery.seed_hosts: [{{ SEED_HOSTS }} ]
zookeeper.connect={{ ZK_NODES }}
```

### 7.2 条件判断

```jinja2
{% if groups['kafka']|length > 1 %}
num.partitions=3
{% else %}
num.partitions=1
{% endif %}

{% if upgrade_cluster | default('') != 'upgrade' %}
cluster.initial_master_nodes: [{{ TEMP_NODE_NAMES }} ]
{% endif %}

{% if NODE_ROLES is defined and NODE_ROLES != 'master' %}
node.roles: [ {{ NODE_ROLES }} ]
{% endif %}
```

### 7.3 复杂条件嵌套

```jinja2
{% if PARAM_KAFKA_AUTH is defined and (PARAM_KAFKA_AUTH == "PLAIN" or PARAM_KAFKA_AUTH == "SCRAM-SHA-256") %}
  {% if PARAM_KAFKA_SSL is defined and PARAM_KAFKA_SSL == 1 %}
listeners=SASL_SSL://{{ inventory_hostname }}:9092
  {% else %}
listeners=SASL_PLAINTEXT://{{ inventory_hostname }}:9092
  {% endif %}
{% endif %}
```

### 7.4 过滤器

```jinja2
{{ var | default('') }}        # 默认值
{{ var | default(0) }}
{{ var | int }}                # 转为整数
{{ groups['kafka'] | length }} # 列表长度
{{ var is defined }}           # 检查是否定义
{{ var is not defined }}
```

---

## 八、特殊变量速查

| 变量 | 含义 | 示例 |
|------|------|------|
| `inventory_hostname` | 当前主机在清单中的名称 | `10.21.18.151` |
| `groups['group_name']` | 组内主机列表 | `groups['kafka'][0]` |
| `groups['group']\|length` | 组内主机数量 | 判断单机/集群 |
| `hostvars[host]['key']` | 访问其他主机的变量 | `hostvars[inventory_hostname]['hostname']` |
| `ansible_facts['services']` | 服务状态字典 | 判断服务是否安装 |
| `ansible_distribution` | 操作系统发行版 | `"Kylin Linux Advanced Server"` |
| `ansible_architecture` | CPU 架构 | `"aarch64"` / `"amd64"` |
| `playbook_dir` | playbook 所在目录 | 构造相对路径 |

---

## 九、Tags（标签）使用模式

Tags 用于选择性执行 playbook 中的部分任务：

```yaml
# 在 task 级别
- name: "启动 kafka"
  shell: systemctl restart asap_kafka
  tags:
    - add_zookeeper
    - add_kafka
    - refresh_config

# 在 play 级别
- hosts: mysql
  tags: mysql          # ← 整个 play 的默认标签
  tasks:
    - name: starting mysql
      service: name=asap_mysql state=started
```

执行时通过 `-t` / `--skip-tags` 控制：

```bash
ansible-playbook -t add_kafka playbooks/add/add-kafka-tag.yml        # 只执行 add_kafka 标签
ansible-playbook -t refresh_config playbooks/kafka/kafka.yml          # 只刷新配置
ansible-playbook --skip-tags=port,uid,time playbooks/precheck/base.yaml  # 跳过检查
```

---

## 十、项目设计模式

### 10.1 Shell 调用 Ansible 模式

```bash
# Shell 脚本作为入口，Ansible 作为执行引擎
function exec_ansible() {
    cd /data/maxs-ops
    ansible-playbook "$@"
    if [ $? -ne 0 ]; then
        exit 1
    fi
}

exec_ansible playbooks/k8s/03.etcd.yml
exec_ansible -t refresh_config playbooks/kafka/kafka.yml
```

### 10.2 Playbook + Role 分离

- **playbooks/**：定义执行顺序和拓扑（hosts、tags、roles 引用）
- **roles/**：定义具体操作和模板（tasks、templates、vars）

```yaml
# playbooks/es/es.yml（顶层编排）
- hosts: es
  roles:
    - es
  gather_facts: False

# roles/es/tasks/main.yml（具体实现）
- name: "分发 es 配置"
  template:
    dest: "{{ item.dest }}"
    src: "{{ item.src }}"
  with_items:
    - { src: "asap_es.service.j2", dest: "/etc/systemd/system/asap_es.service" }
```

### 10.3 Tags 分类模式

三种 Tags 贯穿整个项目：

| Tag | 用途 |
|-----|------|
| `add_<service>` | 首次部署任务 |
| `refresh_config` | 配置刷新（修改配置文件后） |
| `<service_name>` | 启动/停止等运维操作 |

### 10.4 Callback Plugin（task_timer.py）

自定义 `stdout_callback` 插件，在 ansible.cfg 中启用：

```python
class CallbackModule(DefaultCallbackModule):
    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'stdout'
    CALLBACK_NAME = 'task_timer'

    def v2_playbook_on_task_start(self, task, is_conditional):
        # 记录 task 开始时间
        self._current_task_start = datetime.now()
        self._display.display("[TASK START] {}".format(name))

    def _maybe_end_current_task(self):
        # 打印 task 耗时
        elapsed = (end_time - self._current_task_start).total_seconds()
        self._display.display("[TASK END] {} | 耗时: {:.2f}秒".format(name, elapsed))
```

---

## 十一、常用代码片段

### 检查组件是否已安装

```yaml
- name: "判断是否已安装"
  stat:
    path: "{{ es_dir }}"
  register: srv_status

- name: "未安装时执行安装"
  import_role:
    name: es
  when: srv_status.stat.exists == False
```

### 条件判断单机 vs 集群

```yaml
# 集群条件下执行
when: "groups['redis']|length > 1"

# 单机条件下执行
when: groups['vastbase'] | length == 1

# 集群且多条件
when:
  - 'CLICKHOUSE_OPEN_FLAG == "1" and groups["clickhouse"]|length > 1'
  - "(DEL_FLAG | default(0)) != 1"
```

### 服务启动 + 状态确认

```yaml
- name: "开启服务"
  shell: |
    systemctl daemon-reload && systemctl restart asap_es
    systemctl enable asap_es
  ignore_errors: true

- name: "轮询确认服务启动"
  shell: "systemctl is-active asap_es.service"
  register: svc_status
  until: '"active" in svc_status.stdout'
  retries: 10
  delay: 3
```

### 在特定节点执行逻辑

```yaml
# 只在组的第一个节点执行
when: "inventory_hostname == groups.kafka[0]"

# 委托到 oper_master 执行
delegate_to: "{{ groups['oper_master'][0] }}"

# 在本机执行（不 SSH）
connection: local

# 以特定用户执行
become: yes
become_user: vastbase
```

### 从命令输出提取变量

```yaml
- name: 获取内存信息
  shell: free -m | awk 'NR==2{print $2}'
  register: mem

- name: 使用结果
  debug:
    msg: "内存大小: {{ mem.stdout }}"
```
