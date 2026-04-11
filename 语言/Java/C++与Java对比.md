以下是针对 **C++ 开发者快速入门 Java** 的核心差异总结，聚焦基础语法、重要模块和常用工具：

---

## 一、基础语法核心差异

### 1. 程序结构

| 特性 | C++ | Java |
|------|-----|------|
| **入口函数** | 自由定义 | 必须是 `public static void main(String[] args)` |
| **代码组织** | 头文件 + 源文件分离 | 无头文件，一个 `.java` 文件一个公共类 |
| **命名空间** | `namespace` | `package` + 目录结构强制对应 |
| **编译执行** | 编译为机器码直接运行 | 编译为字节码，JVM 解释/JIT 执行 |

```java
// Java：文件名必须和公共类名一致（Hello.java）
package com.example;  // 对应目录 com/example/

public class Hello {  // 公共类名 = 文件名
    public static void main(String[] args) {  // 固定签名
        System.out.println("Hello");  // 无 printf，String 不是 char*
    }
}
```

### 2. 数据类型与变量

| 特性 | C++ | Java |
|------|-----|------|
| **基本类型** | `int`, `char`, `bool` 等 | 类似，但 `bool` → `boolean`，`char` 是 16 位 Unicode |
| **字符串** | `std::string` 或 `char*` | `String` 对象，不可变，用双引号 |
| **数组** | 栈/堆均可 | 只能在堆上，`int[] arr = new int[10]` |
| **常量** | `const` / `constexpr` | `final`（修饰引用时含义不同） |

```java
// Java 关键差异
final StringBuilder sb = new StringBuilder();  
sb.append("a");  // ✓ 可以！final 只锁引用，不锁对象内容

String s1 = "hello";
String s2 = "hello";
s1 == s2;        // true（字符串常量池，类似 C++ 字符串驻留）
s1.equals(s2);   // true（内容比较，始终用这个）

// 数组
int[] arr = {1, 2, 3};           // 简写
int[] arr2 = new int[]{1, 2, 3}; // 完整写法
int[][] matrix = new int[3][4];   // 二维数组，不是连续内存
```

### 3. 面向对象

| 特性        | C++                  | Java                                    |
| --------- | -------------------- | --------------------------------------- |
| **继承**    | `class B : public A` | `class B extends A`                     |
| **多重继承**  | 支持                   | **不支持**（用 `interface` 实现多态）             |
| **虚函数**   | `virtual` 关键字        | 所有非静态方法默认虚函数                            |
| **析构函数**  | `~Class()` 确定性调用     | `finalize()` 不推荐，用 `try-with-resources` |
| **运算符重载** | 支持                   | **不支持**（`+` 对 String 是内置特例）             |

```java
// Java 类定义
public class Animal {
    private String name;           // 默认访问：包内，不是 private
    
    public Animal(String name) {   // 构造器
        this.name = name;          // this 替代 this->
    }
    
    public void speak() {          // 默认 virtual，无需标记
        System.out.println("...");
    }
}

// 继承 + 接口（替代 C++ 多重继承）
interface Flyable {
    void fly();  // 默认 public abstract
}

class Bird extends Animal implements Flyable {
    @Override  // 标记重写，类似 C++ override（C++11）
    public void speak() {
        System.out.println("Chirp");
    }
    
    @Override
    public void fly() { }
}
```

### 4. 内存管理（最大差异）

| C++ | Java |
|-----|------|
| `new` / `delete` 手动管理 | `new` 创建，**自动垃圾回收（GC）** |
| 智能指针 `unique_ptr/shared_ptr` | 只有引用，无指针算术 |
| 栈对象自动析构 | 无栈对象，所有对象在堆上（基本类型除外） |
| RAII 惯用法 | `try-with-resources` / `AutoCloseable` |

```java
// Java：无需 delete，但注意引用赋值是浅拷贝
Object obj = new Object();
obj = null;  // 原对象可被 GC，但时机不确定

// 资源管理（对比 C++ RAII）
try (FileInputStream fis = new FileInputStream("file.txt")) {
    // 自动关闭，类似 C++ unique_ptr 自定义 deleter
} catch (IOException e) {
    e.printStackTrace();  // 异常必须处理或声明
}
```

### 5. 泛型 vs 模板

| C++ 模板 | Java 泛型 |
|----------|-----------|
| 编译期代码生成 | **类型擦除**（运行时无泛型信息） |
| `template <typename T>` | `<T>` |
| 可特化、偏特化 | 不支持 |
| 基本类型可直接用 | 必须用包装类 `Integer` 而非 `int` |

```java
// Java 泛型（类型擦除）
List<Integer> list = new ArrayList<>();  // <> 类似 C++ <>
List rawList = list;  // 可以！但丢失类型安全（C++ 会报错）

// 通配符（C++ 无直接对应）
List<? extends Number> numbers;  // 上界，类似 ? super T 是下界
```

---

## 二、Java 重要模块（对比 C++ 标准库）

| 模块 | Java 包 | 对比 C++ |
|------|---------|----------|
| **集合框架** | `java.util.*` | 替代 STL：`ArrayList`→`vector`, `HashMap`→`unordered_map` |
| **并发** | `java.util.concurrent.*` | 比 `<thread>` + `<mutex>` 更丰富：线程池、原子类、并发集合 |
| **IO/NIO** | `java.io.*`, `java.nio.*` | 替代 `<fstream>` + `<filesystem>`，NIO 支持异步 |
| **网络** | `java.net.*` | 替代 `<sys/socket.h>` 或 Boost.Asio |
| **日期时间** | `java.time.*` (Java 8+) | 替代 `<chrono>`，设计更合理 |
| **函数式编程** | `java.util.function.*`, Stream API | 类似 C++20 ranges + lambda |

### 关键类快速对照

| C++ | Java |
|-----|------|
| `std::vector<T>` | `ArrayList<T>` |
| `std::list<T>` | `LinkedList<T>` |
| `std::map<K,V>` | `HashMap<K,V>` (无序) / `TreeMap<K,V>` (有序) |
| `std::set<T>` | `HashSet<T>` / `TreeSet<T>` |
| `std::unique_ptr<T>` | 无，直接引用赋值，GC 处理 |
| `std::shared_ptr<T>` | 无直接对应，对象引用即共享 |
| `std::thread` | `Thread` 类 / `ExecutorService` |
| `std::mutex` | `synchronized` / `ReentrantLock` |

---

## 三、常见工具链

### 1. 构建工具（对比 CMake）

| 工具 | 用途 | 特点 |
|------|------|------|
| **Maven** | 项目构建 + 依赖管理 | XML 配置，约定优于配置，企业主流 |
| **Gradle** | 同上 | Groovy/Kotlin DSL，更灵活，Android 默认 |

```xml
<!-- Maven pom.xml 示例：对比 CMakeLists.txt -->
<dependencies>
    <!-- 一行引入，自动下载传递依赖 -->
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>32.1.0-jre</version>
    </dependency>
</dependencies>
```

### 2. IDE（对比 Visual Studio/CLion）

| IDE | 特点 | 推荐度 |
|-----|------|--------|
| **IntelliJ IDEA** | 代码补全、重构最强，有免费社区版 | ⭐⭐⭐⭐⭐ |
| **Eclipse** | 老牌免费，插件丰富 | ⭐⭐⭐ |
| **VS Code** | 轻量，需装插件 | ⭐⭐⭐ |

### 3. 调试与监控

| 工具 | 对比 C++ |
|------|----------|
| **JVisualVM / JConsole** | 类似 Valgrind + perf，监控内存、线程、GC |
| **JUnit 5** | 对比 Google Test，单元测试框架 |
| **Mockito** | 对比 gMock，Mock 框架 |
| **SLF4J + Logback** | 对比 spdlog/glog，日志框架 |

### 4. Web 开发核心（如需要）

| 框架 | 说明 | 对比 C++ |
|------|------|----------|
| **Spring Boot** | 企业级 Web 开发标准 | 对比 Crow/ Pistache/ Drogon，但功能远超 |
| **JPA/Hibernate** | ORM 框架 | 对比 SOCI/ODB |
| **Tomcat/Jetty** | Servlet 容器 | 对比自己嵌入的 HTTP 服务器 |

---

## 四、C++ 开发者易犯错误

```java
// 错误1：以为 == 比较内容
String a = new String("hi");
String b = new String("hi");
if (a == b) { }  // false！用 a.equals(b)

// 错误2：试图用指针思维
Object obj;
obj.method();  // 编译错误！Java 引用必须初始化，无空指针（语法上）

// 错误3：忘记异常处理
// C++ 可不捕获，Java 受检异常必须处理或声明 throws

// 错误4：泛型数组
List<Integer>[] array = new List<Integer>[10];  // 编译错误！类型擦除导致

// 错误5：静态绑定思维
// Java 所有方法默认虚函数，无需担心切片问题，但性能略低
```

---
