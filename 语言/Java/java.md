## 1. 安装
```bash
## 安装
# 如果只安装了 JRE，先移除
sudo apt remove openjdk-*-jre
# 安装完整的 JDK（以 Java 21 为例）
sudo apt install openjdk-21-jdk -y
# 或者安装 Java 17
sudo apt install openjdk-17-jdk -y

## 选择版本
# 查看可用的 Java 版本
sudo update-alternatives --config java
sudo update-alternatives --config javac
# 设置 JAVA_HOME
echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$PATH:$JAVA_HOME/bin' >> ~/.bashrc
source ~/.bashrc
# 设置 Java 21 为默认版本（如果多个版本）
sudo update-alternatives --set java /usr/lib/jvm/java-21-openjdk-amd64/bin/java
sudo update-alternatives --set javac /usr/lib/jvm/java-21-openjdk-amd64/bin/javac

## 验证安装
# 验证 JDK 安装
java --version
javac --version
# 查看 JDK 安装路径
echo $JAVA_HOME
ls $JAVA_HOME/bin/ | grep javac

```
## 2. 目录结构
### 1）目录结构
1. 常见的Maven工程目录结构
```
my-project/
├── src/
│   ├── main/
│   │   ├── java/              ← 源代码（.java 文件）
│   │   │   └── com/example/app/
│   │   │       ├── Main.java
│   │   │       └── service/
│   │   └── resources/         ← 主代码资源文件（配置文件、静态资源）
│   │       ├── application.properties
│   │       └── logback.xml
│   │
│   └── test/
│       ├── java/              ← 测试代码（单元测试、集成测试）
│       │   └── com/example/app/
│       │       └── MainTest.java
│       └── resources/         ← 测试资源文件
│           └── application-test.properties
│
├── target/ 或 build/          ← 编译输出目录（自动生成）
│   ├── classes/               ← 编译后的 .class 文件
│   ├── test-classes/          ← 测试代码编译结果
│   └── my-project-1.0.jar     ← 打包后的产物
│
├── pom.xml 或 build.gradle    ← 构建工具配置
└── README.md
```

2. War结构(Web应用)
```
src/main/
├── java/
├── resources/
└── webapp/              ← 替代 resources，存放 WEB 资源
    ├── WEB-INF/
    │   └── web.xml
    ├── index.html
    └── static/
        ├── css/
        └── js/
```
3. 多模块项目
```
parent-project/
├── pom.xml              ← 父 POM，统一管理依赖版本
├── common-module/       ← 公共工具模块
├── service-module/      ← 业务逻辑模块  
├── web-module/          ← Web 入口模块
└── integration-test/    ← 集成测试模块
```
4. Spring Boot典型结构
```
src/main/java/com/example/demo/
├── DemoApplication.java          ← 启动类（根包）
├── config/                        ← 配置类
├── controller/                    ← API 控制器层
├── service/                       ← 业务逻辑层
├── repository/ 或 mapper/          ← 数据访问层
├── entity/ 或 domain/ 或 model/    ← 实体/领域模型
├── dto/                           ← 数据传输对象
└── util/ 或 common/               ← 工具类

src/main/resources/
├── application.yml                ← 主配置
├── application-dev.yml            ← 开发环境配置
├── application-prod.yml           ← 生产环境配置
├── static/                        ← 静态资源（可直接访问）
└── templates/                     ← 模板文件（Thymeleaf等）
```
**关键理解要点**：
1. **包命名规范**：`com.公司名.项目名.模块名`，全小写，避免默认包
    
2. **资源加载路径**：`resources/` 下的文件编译后都在 classpath 根目录，通过 `getResourceAsStream("/config.properties")` 加载
    
3. **测试独立性**：测试代码不应依赖外部服务，应能独立运行（单元测试）或使用内存数据库（集成测试）
    
4. **构建目录忽略**：`target/` 或 `build/` 必须加入 `.gitignore`
### 2）Idea快捷键
| 功能    | Windows/Linux    | Mac             |
| ----- | ---------------- | --------------- |
| 智能提示  | `Ctrl + Space`   | `Ctrl + Space`  |
| 自动导包  | `Alt + Enter`    | `Opt + Enter`   |
| 格式化代码 | `Ctrl + Alt + L` | `Cmd + Opt + L` |
| 查找类   | `Ctrl + N`       | `Cmd + O`       |
| 全局搜索  | `Shift + Shift`  | `Shift + Shift` |
| 生成代码  | `Alt + Insert`   | `Cmd + N`       |


## 基础语法
对比C++  : [[C++与Java对比]]
### 注解
### AOP



## 重要模块
### 并发模型

### 内存模型


## JVM

## 生态框架

## 常见工具

## 入门项目
## 项目的部署
[[一体化部署 vs 分离部署]]