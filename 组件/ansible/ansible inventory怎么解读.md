## INI格式

``` ini
# 单个主机
web1.example.com
web2.example.com

# 分组
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com
db2.example.com

# 分组嵌套（子组）
[servers:children]
webservers
dbservers

# 主机变量
[webservers]
web1.example.com ansible_user=root ansible_port=22

# 组变量
[webservers:vars]
http_port=80
max_clients=200
```

**关键概念**

| 概念               | 说明                   |
| ---------------- | -------------------- |
| **主机(Host)**     | 被管理的远程服务器            |
| **组(Group)**     | 主机的逻辑集合，方便批量操作       |
| **变量(Vars)**     | 主机或组的配置参数 (分为内置和自定义) |
| **子组(Children)** | 组的嵌套，实现层级管理          |

**常用命令验证**

```sh
# 列出所有主机
ansible-inventory --list

# 查看特定组的主机
ansible-inventory --graph webservers

# 测试连通性
ansible all -m ping

# 针对特定组执行命令
ansible webservers -a "uptime"
```