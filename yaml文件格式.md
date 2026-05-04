YAML（YAML Ain't Markup Language）是一种**人类可读的数据序列化格式**，以简洁、易读为核心设计目标。

---

## 一、核心规则

| 规则 | 说明 |
|------|------|
| **大小写敏感** | `Name` 和 `name` 是不同的键 |
| **缩进表示层级** | 用空格（不能用 Tab），通常2个空格 |
| **冒号+空格** | `key: value`，冒号后必须有空格 |
| **井号注释** | `# 这是注释` |

---

## 二、三种基本数据类型

### 1. 标量（Scalar）—— 单个值
```yaml
name: Alice               # 字符串（无需引号）
age: 30                   # 整数
active: true              # 布尔值
salary: 5000.50           # 浮点数
description: "包含: 冒号的字符串"  # 需要引号的情况
multiline: |               # 保留换行
  第一行
  第二行
```

### 2. 列表（List/Sequence）—— 用 `-`
```yaml
fruits:
  - apple
  - banana
  - orange

# 行内写法
numbers: [1, 2, 3]
```

### 3. 字典/映射（Mapping）—— 键值对
```yaml
user:
  name: Bob
  email: bob@example.com

# 行内写法
config: { host: localhost, port: 8080 }
```

---

## 三、常见组合结构

```yaml
# 列表套字典（最常见于配置文件）
servers:
  - name: web01
    ip: 192.168.1.10
    ports:
      - 80
      - 443
  
  - name: db01
    ip: 192.168.1.20
    ports:
      - 3306

# 字典套列表
project:
  name: MyApp
  dependencies:
    - flask
    - requests
    - sqlalchemy
```

---

## 四、特殊语法

| 语法 | 用途 | 示例 |
|------|------|------|
| `|` | 保留换行符 | 多行文本 |
| `>` | 折叠换行（变空格） | 段落文本 |
| `&name` / `*name` | 锚点和引用（复用） | `&defaults` `*defaults` |
| `!!type` | 强制类型 | `!!str 123` |
| `---` | 文档开始分隔 | 多文档文件 |
| `...` | 文档结束标记 | 可选 |

**锚点引用示例：**
```yaml
defaults: &defaults      # 定义锚点
  adapter: postgres
  host: localhost

development:
  <<: *defaults           # 合并引用
  database: dev_db

production:
  <<: *defaults
  host: prod.server.com   # 覆盖特定值
```

---

## 五、与 JSON 对比

| 特性 | YAML | JSON |
|------|------|------|
| 可读性 | 高（适合人类） | 低（适合机器） |
| 语法 | 缩进敏感，简洁 | 大括号/引号冗余 |
| 注释 | 支持 `#` | 不支持 |
| 数据类型 | 自动推断 | 需引号包裹字符串 |
| 适用场景 | 配置文件、DevOps | 数据交换、API |

**同一数据的对比：**
```yaml
# YAML
users:
  - name: Alice
    roles: [admin, user]
```

```json
// JSON
{
  "users": [
    {
      "name": "Alice",
      "roles": ["admin", "user"]
    }
  ]
}
```

---

## 六、一句话总结

> **YAML = 缩进表层级 + 冒号键值对 + 横线表列表，专为配置文件设计的简洁语法。**