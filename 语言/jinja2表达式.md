Jinja2 表达式是一种用于**模板渲染**的语法，让你可以在文本中嵌入动态逻辑。它最初是 Python 的模板引擎，现在被广泛用于 Ansible、Flask、Django 等工具中。

---

## 一、核心概念：双大括号 `{{ }}`

所有 Jinja2 表达式都包裹在 `{{ }}` 中，用于**输出变量或计算结果**：

```jinja2
Hello, {{ username }}!           {# 输出变量 #}
Today is {{ 2024 + 1 }}          {# 输出 2025 #}
```

---

## 二、常见语法速查

### 1. 变量与过滤器（Filters）

用 `|` 管道符处理数据：

```jinja2
{{ name | upper }}              {# 转大写 #}
{{ items | length }}            {# 获取列表长度 #}
{{ price | default(0) }}        {# 设置默认值 #}
{{ html | safe }}               {# 标记为安全HTML #}
{{ users | selectattr("active") }}  {# 筛选属性 #}
```

### 2. 逻辑控制

```jinja2
{# 条件判断 #}
{% if user.age >= 18 %}
  成年人
{% else %}
  未成年人
{% endif %}

{# 循环遍历 #}
{% for item in items %}
  {{ loop.index }}. {{ item.name }}
{% endfor %}

{# 设置变量 #}
{% set total = price * quantity %}
```

### 3. 测试（Tests）

用 `is` 做条件判断：

```jinja2
{% if user is defined %}        {# 变量是否存在 #}
{% if items is none %}            {# 是否为 None #}
{% if name is string %}           {# 是否为字符串 #}
{% if ports is iterable %}        {# 是否可迭代 #}
```

---

## 三、在 Ansible 中的实际应用（最常见场景）

```yaml
# playbook.yml
- name: 部署应用
  hosts: webservers
  vars:
    app_name: myapp
    version: "1.2.3"
    ports: [80, 443]
  
  tasks:
    # 变量插值
    - name: 创建目录
      file:
        path: "/opt/{{ app_name }}"
        state: directory
    
    # 过滤器：获取第一个端口
    - name: 显示主端口
      debug:
        msg: "主端口是 {{ ports | first }}"
    
    # 条件判断
    - name: 安装 Nginx
      apt:
        name: nginx
      when: ansible_os_family == "Debian"
    
    # 循环 + 变量计算
    - name: 创建虚拟主机配置
      template:
        src: vhost.conf.j2
        dest: "/etc/nginx/sites-available/{{ item.name }}"
      loop: "{{ virtual_hosts }}"
      when: item.enabled | default(true) | bool
```

**模板文件 `vhost.conf.j2`：**
```nginx
server {
    listen {{ item.port | default(80) }};
    server_name {{ item.domain }};
    
    {% if item.ssl | default(false) %}
    ssl_certificate {{ item.ssl_cert }};
    {% endif %}
    
    location / {
        proxy_pass http://{{ app_name }}_backend;
    }
}
```

---

## 四、与 Shell/YAML 的区别

| 场景 | Shell | YAML | Jinja2 |
|------|-------|------|--------|
| 变量 | `$var` 或 `${var}` | `key: value` | `{{ var }}` |
| 条件 | `if [ ... ]; then` | 无原生支持 | `{% if ... %}` |
| 循环 | `for i in ...` | 无原生支持 | `{% for ... %}` |
| 过滤器 | 管道 `\|` 是进程管道 | 无 | `\|` 是数据转换 |
| 用途 | 系统命令 | 数据结构 | 动态模板渲染 |

---

## 五、一句话总结

> **Jinja2 表达式 `{{ }}` 用于在模板中插入动态值，`{% %}` 用于控制逻辑（条件/循环），`|` 用于数据转换。**

它是 Ansible 自动化运维的核心技能，让静态的 YAML 配置变得动态灵活。