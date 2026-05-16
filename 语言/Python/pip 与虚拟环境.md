# pip 与虚拟环境

## pip 安装路径

```bash
# 查看 pip 安装路径
pip show <package>

# 查看 pip 位置
which pip
python -m site

# 用户安装目录
python -m site --user-site
```

### 多版本 Python 的 pip

```bash
# Python3 对应的 pip
pip3

# 指定版本
python3.11 -m pip install <package>
```

## 虚拟环境

### venv (Python3 内置)

```bash
# 创建虚拟环境
python -m venv venv_name

# 激活 (Linux/Mac)
source venv_name/bin/activate

# 激活 (Windows)
venv_name\Scripts\activate

# 退出
deactivate
```

### 虚拟环境目录结构

```
venv_name/
├── bin/               # Linux/Mac (或 Scripts/ Windows)
│   ├── python          # 指向系统 Python
│   ├── pip
│   └── activate
├── lib/
│   └── python3.x/
│       └── site-packages/  # 第三方包安装位置
│           ├── some_package/
│           └── ...
└── pyvenv.cfg
```

### requirements.txt

```bash
# 导出依赖
pip freeze > requirements.txt

# 安装依赖
pip install -r requirements.txt

# 最佳实践: 使用 pip-tools
pip install pip-tools
pip-compile requirements.in  # 生成 requirements.txt
pip-sync requirements.txt    # 同步安装
```

### 虚拟环境隔离原理

- 每个虚拟环境有**独立的** `site-packages/`
- 系统 Python 本身不被修改
- 激活时 `PATH` 变量被修改，优先找到 venv 的 bin

### 位置对比

| 环境 | 第三方包位置 |
|------|-------------|
| 系统 Python | `/usr/lib/python3.x/site-packages/` |
| 用户安装 | `~/.local/lib/python3.x/site-packages/` |
| 虚拟环境 | `./venv/lib/python3.x/site-packages/` |

## 与 Java 对比

Java 无虚拟环境概念，依赖统一由 Maven/Gradle 管理在项目本地 (`~/.m2/` 为本地仓库)。

```java
// Java: Maven 管理依赖 (pom.xml)
// <dependency>
//     <groupId>com.example</groupId>
//     <artifactId>artifact</artifactId>
//     <version>1.0</version>
// </dependency>

// Python: pip 安装 (requirements.txt 或 pyproject.toml)
// pip install requests==2.28.0
```